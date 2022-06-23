
## Filter Block == Meta block
> 필터 블록은 SST의 블록입니다. 필터 블록은 여러 필터로 구성되며 각 데이터 블록은 필터에 해당합니다(하지만 하나의 필터는 여러 데이터 블록에 해당할 수 있음).즉, 이러한 data block의 데이터는 모두 이 필터에 있습니다. 따라서 필터의 수는 데이터 블록의 수보다 작거나 같아야 합니다.

> offset i는 SST 파일에 i번째 필터의 오프셋을 저장합니다. offset list offset에 존재 하는것은 offset list 가 SSTfile에서의  offset。

> 읽을시에는 먼저 datablock의 filter를 검색(key)

![](https://www.huliujia.com/images/leveldb_SST_filter_block%E6%A0%BC%E5%BC%8F.png)

```c++
//table/filter_block.h
class FilterBlockBuilder {
public:
// explicit 키워드는 자신이 원하지 않은 형변환이 일어나지 않도록 제한하는 키워드이다.
  explicit FilterBlockBuilder(const FilterPolicy*);

  FilterBlockBuilder(const FilterBlockBuilder&) = delete;
  FilterBlockBuilder& operator=(const FilterBlockBuilder&) = delete;


  void StartBlock(uint64_t block_offset);
  void AddKey(const Slice& key);
  Slice Finish();

private:
  void GenerateFilter();

  const FilterPolicy* policy_; //filter policy，like BloomFilter
  
  std::string keys_; //filter list 
  std::vector<size_t> start_; //key(filter) 위치 list

  std::string result_; //filterlist 의 buffer
  std::vector<Slice> tmp_keys_; //임시 저장하고 입력한 key
  std::vector<uint32_t> filter_offsets_; //filter의 offset
};



class FilterBlockReader {
 public:
  // REQUIRES: "contents" and *policy must stay live while *this is live.
  FilterBlockReader(const FilterPolicy* policy, const Slice& contents);
  bool KeyMayMatch(uint64_t block_offset, const Slice& key);

 private:
  const FilterPolicy* policy_;
  const char* data_;    // Pointer to filter data (at block-start)
  const char* offset_;  // Pointer to beginning of offset 배열  
  size_t num_;          // Number of entries in offset array (크기 )
  size_t base_lg_;      // Encoding parameter (see kFilterBaseLg in .cc file)
};

```
---

```c++
//table/filter_block.cc
//slice는 string이랑 같지만 값이 싸다
void FilterBlockBuilder::AddKey(const Slice& key) 
{
  Slice k = key;
  start_.push_back(keys_.size());   // 들어온 key의 크기 만큼 ; start_
  keys_.append(k.data(), k.size()); // 키를 순서대로 넣기 
}

```
---
```c++
//table/filter_block.cc
//StartBlock의 임무는 block_offset을 기준으로 필터를 생성하는 것인데, 필터의 개수는 블록의 개수와 관련이 없고 block_offset에만 관련이 있습니다. 기본적으로 필터는 2K block_offset마다 생성됩니다. 2K의 기본값은 상수 kFilterBase에 의해 지정됩니다.

void FilterBlockBuilder::StartBlock(uint64_t block_offset) { // block 위치가 입력 값
  uint64_t filter_index = (block_offset / kFilterBase); // filter 수량은 위치 / 크기
  // filter_offsets_size도 filter 수량
  assert(filter_index >= filter_offsets_.size()); // filter_index가
  while (filter_index > filter_offsets_.size()) {
    GenerateFilter(); // filter 생성
  }

//예를 들어 block_offset이 5K이면 filter_index는 2이고 그러면 두 개의 필터가 필요합니다. 다음 호출의 block_offset이 5.5K이면 계산된 filter_index는 여전히 2이고 새 필터가 생성되지 않습니다. 검색할 때 block_offset을 2K로 나누어 filter_offsets_에서 필터의 위치를 ​​얻은 다음 filter_offsets_에 따라 filter를 찾습니다.

}
```
----

```c++
// GenerateFilter는 keys_ 및 start_에 저장된 키 목록으로 필터를 생성하고 생성 후 keys_ 및 start_를 지웁니다.

void FilterBlockBuilder::GenerateFilter()
{
  const size_t num_keys = start_.size();
  //Step1
  if ( num_keys == 0 )
  {
    filter_offsets_.push_back(result_.size());
    return;
  }

  //Step2
  start_.push_back(keys_.size());
  tmp_keys_.resize(num_keys);
  for ( size_t i = 0; i < num_keys; i++ )
  {
    const char *base = keys_.data() + start_[i];
    size_t length = start_[i + 1] - start_[i];
    tmp_keys_[i] = Slice(base, length);
  }

  //Step3
  filter_offsets_.push_back(result_.size());
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  //Step4
  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}

// 1단계: num_keys= start_size()= 키의 사이즈 가 0이면 다음 필터의 오프셋을 filter_offsets_에 넣습니다. 즉, 필터 오프셋이 다음 필터를 가리키도록 합니다.


// 2단계: keys_의 키를 구문 분석하고 tmp_keys_에 넣습니다.

//3단계: CreateFilter를 호출하여 필터를 생성하면 새 필터가 result_에 추가됩니다. policy_는 추상 클래스 포인터이고 기본 정책은 BloomFilter입니다.

//4단계: 빌더의 데이터를 재설정하여 다음 필터를 준비합니다.

```

```c++

Slice FilterBlockBuilder::Finish()
{
  if ( !start_.empty())
  {
    GenerateFilter();
  }

  const uint32_t array_offset = result_.size();
  for ( size_t i = 0; i < filter_offsets_.size(); i++ )
  {
    PutFixed32(&result_, filter_offsets_[i]);
  }
  PutFixed32(&result_, array_offset);
  result_.push_back(kFilterBaseLg);
  return Slice(result_);
}

//Finish를 호출하여 필터 블록의 구성을 완료합니다. Finish는 먼저 나머지 키에 대한 새 필터를 생성한 다음 필터 오프셋 목록을 result_로 순서대로 인코딩하고 오프셋 목록의 오프셋을 result_의 끝까지 인코딩합니다. 마지막으로 상수 kFilterBaseLg가 끝에 추가되며 기본값은 11, 2^11=2048=2KB로 필터를 생성할 block_offsets 수를 나타냅니다.

````


![](https://www.huliujia.com/images/leveldb_filter_block%E7%BB%93%E6%9E%84.png)



### . 예 
![image](https://user-images.githubusercontent.com/86946575/175249208-962b89e9-6b58-4bf6-a3de-da5557d26ad3.png)
