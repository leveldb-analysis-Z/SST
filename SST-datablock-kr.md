# SST

> SST 파일은 N개의 데이터 블록, 1개의 필터 블록, 1개의 메타 인덱스 블록, 1개의 인덱스 블록 및 1개의 footer로 구성

데이터 블록, 필터 블록 및 메타인덱스 블록은 동일한 ==  저장 구조


         filter block == meta block 0 ~ k-1
![image](https://www.huliujia.com/images/levedb_SST_%E6%95%B4%E4%BD%93%E6%A0%BC%E5%BC%8F.png)
![](https://img-blog.csdnimg.cn/3488e41219f347a7ae888ab66a1e85fb.png)


## data block
> leveldb/table/block_builder.cc

```c++
//table/block_builder.h
// vector 컨테이너는 자동으로 메모리가 할당되는 배열
// Slice에는 실제 데이터가 없고 데이터에 대한 포인터만 있으며 복사본이 포인터의 복사본일 뿐이므로 오버헤드가 낮습니다.

// 저장: string -> Slice
// 호출: Slice -> string

class BlockBuilder
{
public:
  explicit BlockBuilder(const Options *options);   //options :levelDB의 option들
  void Add(const Slice &key, const Slice &value);  
  Slice Finish();
  void Reset();
  size_t CurrentSizeEstimate() const;
  bool empty() const

private:
  const Options *options_;  //options :levelDB의 기본 option들
  std::string buffer_;     // 데이터 buffer
  std::vector <uint32_t> restarts_;   // restart point
  int counter_;      // 현재 restart point와 관련된 key의 개수
  bool finished_;    // 위의 Finish()함수가 이미 호출되었는지 여부
  std::string last_key_;  // 마지막으로 삽입된 키
};

```
----

```c++
// BlockBuilder::Add
//table/block_builder.cc
// Add 함수를 호출하여 현재 블록에 새로운 k/v 쌍 {key, value}를 추가합니다
// 간단하게 말하자면 assert 함수는 디버깅 모드에서 개발자가 오류가 생기면 치명적일 것이라는 곳에 심어 놓는 에러 검출용 코드입니다.
void BlockBuilder::Add(const Slice &key, const Slice &value)
{
  Slice last_key_piece(last_key_);
  assert(!finished_); //false 면  끝. finished_ =  false
  assert(counter_ <= options_->block_restart_interval); 
  // block_restart_interval = 16 ; restart point  간격 
  assert(buffer_.empty() || options_->comparator->Compare(key, last_key_piece) > 0); 
  size_t shared = 0;

//Step1
// 만약 현재 key개수가 block_restart_interval(restart point 키의 개수 )가 16개(default)을 초과하지 않으면 
// 공통 길이 값(shared) 와  다른 부분 (non_shared) 중 더 작은 것을 

  if ( counter_ < options_->block_restart_interval )  // counter_ < 16
  {
    const size_t min_length = std::min(last_key_piece.size(), key.size()); // 바로 이전에 들온 key size 와 지금 추가한 key size 을 비교해서 적은 것 min_length에 저장
    while ((shared < min_length) && (last_key_piece[shared] == key[shared]))
    {
      shared++;   //이전 키와 들어온 키가 같은 부분이 있으면  shared++ 
    }
  } else     // counter_ > 16  (임계값 초과)
  {
    restarts_.push_back(buffer_.size());  // restart point를 다시 설정하기 위해 지금의 buffer의 크기를 주고 끝
    counter_ = 0;
  // 임계값을 초과하면 현재 restart point 이 비활성화되고 현재 키가 새 재시작 지점으로 설정됩니다.재시작 지점 사이의 간격은 고정되어 있으며 기본적으로 16개의 키가 그룹을 구성하고 그룹의 첫 번째 키는 재시작 지점입니다
  
  }

  const size_t non_shared = key.size() - shared; //같지 않은 부분



  //Step2
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());
  buffer_.append(key.data() + shared, non_shared); //size 나눠서 저장
  buffer_.append(value.data(), value.size());

// shared, non_shared 및 value_size를 각각 버퍼에 씁니다. 



  //Step3
  last_key_.resize(shared); // 공유된 부분을 업데이트
  last_key_.append(key.data() + shared, non_shared); //key  저장
  assert(Slice(last_key_) == key); 
  counter_++;
// last_key를 현재 키로 기준으로 설정하고 카운터에 1을 추가합니다. 즉, 현재 재시작 지점과 연결된 키 수에 1을 추가합니다(재시작 지점 키 자체도 연결된 키임).
// 다시 시작 지점의 논리는 각 그룹의 첫 번째 키의 shared가 0이어야 합니다. 

}


```
----
```c++
// BlockBuilder::Finish
//table/block_builder.cc
//블록 작성을 완료하고 블록 내용을 참조하는 슬라이스를 반환합니다. 반환된 슬라이스는 이 빌더의 수명 동안 또는 Reset()이 호출될 때까지 유효한 상태로 유지됩니다.

Slice BlockBuilder::Finish()
{
  for( size_t i = 0; i < restarts_.size(); i++ ) // 지금 restart point 크기만큼 
  {
    PutFixed32(&buffer_, restarts_[i]); // 버퍼에 저장
  }
  PutFixed32(&buffer_, restarts_.size()); // 크기도 저장 
  finished_ = true;   // add 가 더는 안됨
  return Slice(buffer_);  // 지금까지 추가된 값을 Slice type으로 바꿔서 저장
}

```
----
```c++


// BlockBuilder::Reset: BlockBuilder 재설정, 데이터 지우기, restart point  재설정, Finished_를 false로 설정합니다.
void BlockBuilder::Reset() {
  buffer_.clear();
  restarts_.clear();
  restarts_.push_back(0);  // First restart point is at offset 0
  counter_ = 0;
  finished_ = false;
  last_key_.clear();
}


//BlockBuilder::CurrentSizeEstimate: 버퍼가 차지하는 총 공간 및 restart point 정보(크기 포함)를 반환합니다.

size_t BlockBuilder::CurrentSizeEstimate() const {
  return (buffer_.size() +                       //  data buffer
          restarts_.size() * sizeof(uint32_t) +  // Restart array
          sizeof(uint32_t));                     // Restart array length
}



BlockBuilder::empty: buffer_가 비어 있는지 여부를 반환합니다.

```
Buffer 
![](https://www.huliujia.com/images/leveldb_block%E5%AD%98%E5%82%A8%E6%A0%BC%E5%BC%8F.png)


----



restart point 와 key-value group는 1:1 매핑

restart point i는 SST 파일에서 i번째 key-value group 오프셋을 나타냅니다.

- key-value group에는 X개의key-value 포함됩니다
  - data block은 x = 16
  - index block x = 1
  - 메타인덱스 블록의 경우 전체 블록에는 하나의 키-값만 있고 key는 필터 블록의 이름이고 value은 필터 블록의 오프셋과 크기입니다.
    - 키-값 그룹에서 첫 번째 키가 전체 키를 저장해야 한다는 점을 제외하고 다른 키에 첫 번째 키와 공통 접두사가 있는 경우 키 접미사만 저장됩니다.
    - shared 및 non_shared는 각각 공통 접두사와 키 접미사의 길이를 나타내고, value_size는 값의 크기를 나타냅니다. 세 값 모두 가변 길이 인코딩을 사용하여 저장되므로 점유 공간은 1바이트에서 5바이트까지 다양합니다.


- type: compaction을 하였는지 표시
  - 압축 안하면: type = 0  
  - 압축 하면 :  type = 1 

- crc:block의 오류 검사에 사용
![](https://www.huliujia.com/images/leveldb_SST_X_block%E6%A0%BC%E5%BC%8F.png)
![](https://img-blog.csdnimg.cn/436c3d0d4d1d42abba31e2292ede381a.png)



