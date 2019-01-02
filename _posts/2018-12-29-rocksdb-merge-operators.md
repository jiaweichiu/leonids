---
layout: post
title:  "RocksDB merge operators"
categories: [systems]
comments: false
math: true
excerpt: "Study merge operators in RocksDB."
---

RocksDB is a reliable and efficient embedded key-value store. It can be used to power databases such as MySQL (see MyRocks). RocksDB came from LevelDB but has many additional features. One such feature is "merge operators". The basic idea is that if the "value" is a list or a set, you would not want to keep rewriting the entire list or set each time you insert or delete an element. Instead, you want to keep pushing "merge operands" such that when someone queries for a key, we merge all these operands to arrive at a final result for the given key.

Take for example the "value" being a list of strings. Each "merge operand" can be a list of strings to be appended to the current list of strings. A merge is as simple as concatenating all the merge operands.

Would it not be inefficient if for every query, we need to read lots of list fragments and concatenate them? Actually no. The reason is that during compaction (an important operation of LSM trees), merges can happen such that all these list fragments get concatenated or merged into one.

Things get more tricky when someone holds a snapshot. In this case, we cannot merge everything. Take for example MMMSMMMM where M denotes a merge operand and S denotes a snapshot. In this case, when compaction happens, we cannot merge all the 7 M's because otherwise, our snapshot will be confused as it would not be able to see the result of 3 M's. Instead, the best we can do here is to get M1-S-M2 where M1 is the result of merging MMM and M2 is the result of merging MMMM.

# Case study: Sets of strings

We are going to work through an example where the "value" is a set of strings. For simplicity, we only allow "insert" in this example. If you want "delete", you will need to keep two lists: one for inserted elements and one for deleted elements. The list of deleted elements are like "tomb markers".

For serialization, we will use protocol buffers. Here is the trivial proto definition:
```proto
// protoc merge.proto --cpp_out=./

syntax = "proto3";

message MyList {
  repeated string data = 1;
}
```

Next we have some routine serialization and deserialization utility functions. We will be lazy and not propagate errors because this is just an example not production code.

```cpp
void Deserialize(const Slice &slice, MyList *val) {
  // Ideally, we like a ParseFromStringView to avoid calling slice.ToString
  // which will lead to string copying.
  // Here, we use Boost to create a istream from slice and ParseFromIstream.
  namespace bio = boost::iostreams;
  bio::basic_array_source<char> src(slice.data(), slice.size());
  bio::stream<bio::basic_array_source<char>> stream(src);
  CHECK(val->ParseFromIstream(&stream));
  // The lazy version of the above:
  // CHECK(val->ParseFromString(slice.ToString()));
}

void Serialize(const MyList &val, string *out) {
  CHECK(val.SerializeToString(out));
}
```

Before we have our merge operator, we need some more utilities.
```cpp
void DeserializeInsert(const Slice slice, std::set<string> *out) {
  MyList v;
  Deserialize(slice, &v);
  for (int i = 0; i < v.data_size(); ++i) {
    out->insert(v.data(i));
  }
}

template <class T> MyList BuildListHelper(const T &a) {
  MyList out;
  for (const string &s : a) {
    out.add_data(s);
  }
  return out;
}

auto BuildList = BuildListHelper<vector<string>>;
auto BuildListFromSet = BuildListHelper<std::set<string>>;
```

The first function `DeserializeInsert` deserializes the given slice and insert everything it saw into the given set of strings. The functions `BuildList`, `BuildListFromSet` simply build `MyList` (proto) from a vector of strings or a set of strings.

Now, we have the interesting part: the definition of our merge operator.

```cpp
class AddOperator : public MergeOperator {
public:
  bool FullMergeV2(const MergeOperationInput &merge_in,
                   MergeOperationOutput *merge_out) const override {
    std::set<string> out;
    int existing_size = 0; // For logging.
    if (merge_in.existing_value) {
      DeserializeInsert(*merge_in.existing_value, &out);
      existing_size = out.size();
    }
    for (Slice s : merge_in.operand_list) {
      DeserializeInsert(s, &out);
    }
    MyList out2 = BuildListFromSet(out);
    Serialize(out2, &merge_out->new_value);
    LOG(INFO) << "FullMerge: existing_size=" << existing_size
              << " num_operands=" << merge_in.operand_list.size()
              << " final_list_size=" << out2.data_size();
    return true;
  }

  bool PartialMergeMulti(const Slice &key,
                         const std::deque<Slice> &operand_list,
                         std::string *new_value,
                         Logger *logger) const override {
    std::set<string> out;
    for (Slice s : operand_list) {
      DeserializeInsert(s, &out);
    }
    MyList out2 = BuildListFromSet(out);
    Serialize(out2, new_value);
    LOG(INFO) << "PartialMerge: " << operand_list.size()
              << " operands, final list size " << out2.data_size();
    return true;
  }

  const char *Name() const override { return "AddOperator"; }
};
```

Notice we define two very similar functions. One is a full merge. The other is a partial merge. The full merge may have an "existing value" whereas a partial merge does not. Otherwise, they are the same. We did not like using the ```AssociativeMergeOperator``` because it can only combine two lists at a time and if we need to combining `N` lists, then we may need to do `O(N^2)` work, roughly speaking. We also do some logging to better understand what is happening in our toy example.

Next, we have our actual class `MyLists` that uses RocksDB to keep a key-value store from string to a set of strings. This part is somewhat straightforward.

```cpp
#define CHECK_STATUS(s)                                                        \
  {                                                                            \
    auto status = s;                                                           \
    CHECK(status.ok()) << status.ToString();                                   \
  }

class MyLists {
protected:
  std::shared_ptr<DB> db_;
  WriteOptions write_option_;
  ReadOptions get_option_;

public:
  explicit MyLists(std::shared_ptr<DB> db) : db_(db) { CHECK(db_); }
  virtual ~MyLists() {}

  // For prod code, we return a status or bool. Here, we just want to have fun.
  void Set(const std::string &key, const MyList &value) {
    string v;
    Serialize(value, &v);
    Slice slice(v);
    CHECK_STATUS(db_->Put(write_option_, key, slice));
  }

  void Add(const std::string &key, const MyList &value) {
    string v;
    Serialize(value, &v);
    Slice slice(v);
    CHECK_STATUS(db_->Merge(write_option_, key, slice));
  }

  void Remove(const std::string &key) {
    CHECK_STATUS(db_->Delete(write_option_, key));
  }

  bool Get(const std::string &key, MyList *value) {
    std::string str;
    auto s = db_->Get(get_option_, key, &str);
    if (s.IsNotFound()) {
      value->clear_data();
      return true;
    }
    if (!s.ok()) {
      return false;
    }
    Deserialize(str, value);
    return true;
  }
};
```

## Sample run 1

Finally, let's run our code and see what is happening. Our first run does the following.

1. Set an initial value which is a list of two elements.
1. Push in five merge operands.
1. Do a GET and observe that a full merge involving five operands happened.
1. Do a GET and observe that a full merge involving five operands happened. (So every time you do a GET, you need a full merge, until there is a compaction.)
1. Force a compaction and observe that the same full merge happened.
1. Do a GET and observe that no merging happened because the compaction has done its job. (This point is important!)

```cpp
void Run1(std::shared_ptr<DB> db) {
  MyLists lists(db);
  lists.Set("a", BuildList({"jjj", "iii"}));
  lists.Add("a", BuildList({"hhh", "ggg"}));
  lists.Add("a", BuildList({"fff"}));
  lists.Add("a", BuildList({"eee"}));
  lists.Add("a", BuildList({"ddd"}));
  lists.Add("a", BuildList({"ccc", "bbb", "aaa"}));
  MyList v;
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << "Forcing compaction";
  CompactRangeOptions compact_opt;
  CHECK_STATUS(db->CompactRange(compact_opt, nullptr, nullptr));
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << v.DebugString();
}
```

Here is the output of the first run:
```
I1229 09:15:54.934182 11041 merge_sets.cc:193] Running GET
I1229 09:15:54.934286 11041 merge_sets.cc:116] FullMerge: existing_size=2 num_operands=5 final_list_size=10
I1229 09:15:54.934306 11041 merge_sets.cc:195] Running GET
I1229 09:15:54.934334 11041 merge_sets.cc:116] FullMerge: existing_size=2 num_operands=5 final_list_size=10
I1229 09:15:54.934348 11041 merge_sets.cc:197] Forcing compaction
I1229 09:15:54.942070 11043 merge_sets.cc:116] FullMerge: existing_size=2 num_operands=5 final_list_size=10
I1229 09:15:54.982379 11041 merge_sets.cc:200] Running GET
I1229 09:15:54.982792 11041 merge_sets.cc:202] data: "aaa"
data: "bbb"
data: "ccc"
data: "ddd"
data: "eee"
data: "fff"
data: "ggg"
data: "hhh"
data: "iii"
data: "jjj"
```

## Sample run 2

In our second run, we will create a snapshot in between our merges and observe that we need to do a partial merge. The steps are:

1. Set an initial value which is a list of two elements.
1. Push in just three merge operands.
1. Create a snapshot.
1. Push in two more merge operands.
1. Do a GET and observe that a full merge involving five operands happened.
1. Do a GET and observe that a full merge involving five operands happened.
1. Force a compaction and observe that unlike the previous run, we cannot do a full merge on five merge operands. Instead we see a full merge on three operands and a partial merge on two operands.
1. Do a GET and observe that we have a full merge involving one operand. The one operand is the output of the partial merge: `{aaa,bbb,ccc,ddd}`. The existing value is the output of the initial full merge and is of size 6: `{eee,fff,ggg,hhh,iii,jjj}`.
1. Release snapshot. Maybe do a sleep.
1. Force a compaction. We expect to see the same full merge involving one operand. However, this does not seem to happen. We're not sure why.
1. Do a GET. We actually see a full merge involving one operand.

Here is the code:
```cpp
void Run2(std::shared_ptr<DB> db) {
  MyLists lists(db);
  lists.Set("a", BuildList({"jjj", "iii"}));
  lists.Add("a", BuildList({"hhh", "ggg"}));
  lists.Add("a", BuildList({"fff"}));
  lists.Add("a", BuildList({"eee"}));
  LOG(INFO) << "Creating snapshot";
  auto snapshot = db->GetSnapshot();
  lists.Add("a", BuildList({"ddd"}));
  lists.Add("a", BuildList({"ccc", "bbb", "aaa"}));
  MyList v;
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << "Forcing compaction";
  CompactRangeOptions compact_opt;
  CHECK_STATUS(db->CompactRange(compact_opt, nullptr, nullptr));
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << "Releasing snapshot";
  db->ReleaseSnapshot(snapshot);
  LOG(INFO) << "Forcing compaction again";
  CHECK_STATUS(db->CompactRange(compact_opt, nullptr, nullptr));
  LOG(INFO) << "Running GET";
  lists.Get("a", &v);
  LOG(INFO) << v.DebugString();
}
```

Here are the logs:
```
I1229 09:16:33.344550 11101 merge_sets.cc:211] Creating snapshot
I1229 09:16:33.344672 11101 merge_sets.cc:216] Running GET
I1229 09:16:33.344846 11101 merge_sets.cc:116] FullMerge: existing_size=2 num_operands=5 final_list_size=10
I1229 09:16:33.344892 11101 merge_sets.cc:218] Running GET
I1229 09:16:33.344969 11101 merge_sets.cc:116] FullMerge: existing_size=2 num_operands=5 final_list_size=10
I1229 09:16:33.345005 11101 merge_sets.cc:220] Forcing compaction
I1229 09:16:33.352044 11103 merge_sets.cc:132] PartialMerge: 2 operands, final list size 4
I1229 09:16:33.352134 11103 merge_sets.cc:116] FullMerge: existing_size=2 num_operands=3 final_list_size=6
I1229 09:16:33.392138 11101 merge_sets.cc:223] Running GET
I1229 09:16:33.392293 11101 merge_sets.cc:116] FullMerge: existing_size=6 num_operands=1 final_list_size=10
I1229 09:16:33.392343 11101 merge_sets.cc:225] Releasing snapshot
I1229 09:16:33.392359 11101 merge_sets.cc:227] Forcing compaction again
I1229 09:16:33.392410 11101 merge_sets.cc:229] Running GET
I1229 09:16:33.392484 11101 merge_sets.cc:116] FullMerge: existing_size=6 num_operands=1 final_list_size=10
I1229 09:16:33.392794 11101 merge_sets.cc:231] data: "aaa"
data: "bbb"
data: "ccc"
data: "ddd"
data: "eee"
data: "fff"
data: "ggg"
data: "hhh"
data: "iii"
data: "jjj"
```

We are still puzzled why forcing a compaction after the snapshot is released did not cause a full merge (involving one operand) to happen. We have tried sleeping after releasing the snapshot and also creating more snapshots so that there's two partial merges and two operands instead of one. However, we still observe no full merge when we compact after the release of snapshot(s). We are not sure why. In fact, even if we restart the program and reload the database and force a compaction, we still do not see a full merge happens due to the compaction. It is as if the snapshots are

# Addenum: CompactRange vs CompactFiles
We realize that `CompactRange` only flushes the memtable to L0 SSTables on disk. When we call `CompactRange` the second time, the memtable is empty and the L0 SSTables are small and do not need compaction. Hence, nothing happens. No merge. No compaction.

To force a compaction of the L0 SSTables, after the snapshot is released, we need to use `CompactFiles`. In the main, we can use `FlushedFileCollector` (found in RocksDB unit tests) to get the list of flushed files after the first compaction. Then we can pass these files to `CompactFiles`.

```cpp
Options rdb_opt;
FlushedFileCollector *collector = new FlushedFileCollector();
rdb_opt.listeners.emplace_back(collector);
```

In the actual run:
```cpp
LOG(INFO) << "Forcing compaction via CompactFiles";
// Now that snapshot is released, this should merge the two segments.
vector<string> files = collector->GetFlushedFiles();
db->CompactFiles(CompactionOptions(), files, 0);
```

You will observe the full merge of two segments, one due to `PartialMerge` earlier, the other due to `FullMerge` earlier. Previously, these two segments cannot be merged due to a snapshot being held. After `CompactFiles`, subsequent GET requests do not require any merging.
