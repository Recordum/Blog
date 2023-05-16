CMU mallocLab 과제를 simply segregated storage(단순 분리 저장) 전략으로 구현한 자료는 별로 없어서, 직접 구현한 내용을 정리해보려 한다. implicit-free-list 나 explicit-free-list 에 대한 개념을 이해하고 포스팅을 읽을 것을 추천한다.
* 주의사항
1. __이 포스팅은 32bit system 기준으로 작성되었음을 알려드립니다__
2. __CMU mallocLab에서 제공하는 macro들을 사용하고 있으나, macro에 대한 설명은 따로 포함하지 않습니다.__
* 목차
	1. segregated storage?
	2. simply segregated storage 컨셉
   	3. 구현하기

# segregated storage?

> segreated storage의 기본 컨셉은 가용리스트(free-list)들을 size 별로 분리해 (size-class) 관리하는것을 의미한다._(size-class의 범위는 임의로 정할 수 있으나, 이 포스팅에서는 2의 제곱단위로 같은 size끼리 size-class를 나눌예정이다)_

![](https://velog.velcdn.com/images/mingyu/post/6325b195-feed-43b1-982f-79e1cc60336f/image.png)


즉 같은 size-class 별로 Linked List를 구성하고, 각 Linked List를 배열(혹은 각 Linked List를 가르키는 포인터들의 집합)에 저장한다.

+ 배열을 쓰지 않고싶다면 prologue block안에 size-class 수만큼 block을 추가해 각 Linked List를 가르키는 포인터로 사용할 수 있다. 

위 그림 처럼 size-class에 따라 Linked List를 관리하면 implicit-free-list 나 explicit-free-list 보다 __할당시간을 줄일 수 있다__는 장점이 있다.

explicit-free-list를 예로 들어보면, explicit-free-list는 __모든 free-list__ 중에 할당할 크기와 같은 크기의 free-block 을 탐색해야한다. 

하지만 segregated storage는 size 별로 free-list를 구성하기 때문에, 모든 free-list을 탐색하지 않고, 특정 __size-class의 free-list__만 탐색하면 된다.

# simply segregated storage 컨셉
![](https://velog.velcdn.com/images/mingyu/post/793124ed-a4bb-4666-812d-b522c705ae80/image.png)
> simply segregated storage를 나타내는 그림 (이미지 출처 : [stackoverflow](https://stackoverflow.com/questions/64096093/how-does-simple-segregated-storage-pre-allocate-lists-of-fixed-block-sizes))

simply segregated storage(단순 분리 저장) 는 segregated storage 구현 중 하나이다. 

simply segregated storage(단순 분리 저장)은 __할당시간은 최소로__ 줄이는데 매우 유효하고, 구현이 간단하다. segregated storage 는 전체적인 흐름은 다음과 같다. 


1. heap memory에 할당이 요청될때 free-list를 탐색하고 할당할 free-block이 없으면 Chunksize 만큼 heap size를 늘린다.
![](https://velog.velcdn.com/images/mingyu/post/7c1abc59-ac84-4a7e-a670-074de86f9e19/image.png)

2. 요청한 크기 만큼 늘린 heap size를 요청한 크기 만큼 동일하게 나눈다.
![](https://velog.velcdn.com/images/mingyu/post/0d997979-9a3f-4d28-bd2a-7fad6eac826a/image.png)

3. 동일한 크기로 나눈 free-block들을 Linked List로 만들고 size-class에 맞게 segregated storage에 저장한다.

4. 다시 메모리 할당 요청이오면 1~3 을 반복한다.

위 과정을 살펴보면 알 수 있듯이 simply segregated storage는 free-block 을 분할하고 합치는 과정이 없다! 

이러한 특징들이 simply segregated의 장단점을 부각 시키게 된다.

* 장점
	1. 블록을 할당하고 반환하는 시간이 상수시간으로 매우 빠르다.
   	2. free-block을 분할하거나 합치지 않기 떄문에 block에 overhead를 줄일 수 있다. 
    >footer나 이전 block 을 가르키는 predecessor등이 필요하지 않다. 이후에 나오겠지만 payload를 제외하고 필요한 블록은 1 WORD 크기의 next(succ)포인터 뿐이다.

* 단점
	1. 블록을 합치거나 분할하지도 않기 때문에 __극단적인 내외부 단편화__를 유발한다.



# 구현하기
이 글에서는 free-linked-list를 관리하는 insert,delete 등은 설명하지 않고, simply-segregated 에서만 사용하는 핵심 로직 들을 중점적으로 설명하겠디.

## block 구성(adjust_size)
![](https://velog.velcdn.com/images/mingyu/post/c60e92c6-cd9e-454a-8142-a580b7c580d5/image.png)

block 구성은 위 그림과 같다. 
* free-block :  List에서 next block을 가르키는 포인터를 저장하는 Next block(1word)
* allocated-block : size를 저장하는 Head block(1word)

즉 1word + payload 이므로 최소 블록 사이즈는 2word(8byte)이다. 이에 따라 할당 size를 조정하는 함수는 다음과 같이 구현할 수 있다.

segregated storage에 저장되는 size-class 는 $8*(2^n)$ 임을 유의하자

```code
size_t define_adjust_size(size_t size){
    int n = 0;
    int flag = 0;
    size = size + WSIZE;
    while(1){
        flag = flag + (size % 2);
        size = (size) / 2;
        n += 1;
    }
    return 1<<(n+1);
}
```
## init segregated-free-storage
```code
static unsigned int* segregated_free_list[30];
```
전역 변수로 index 크기가 30인 배열을 선언했다.

크기가 30인 이유는 최소 크기가 8바이트이고, 32bit system 에서 최대크기가 4GB이기 때문(사실 mallocLab은 heapsize를 최대 20mb로 지정하기 때문에 인덱스 크기를 줄여도 상관 없을듯 하다)

* 앞서 말했듯이 배열이 아닌 prologue block에 포인터를 추가하는 방법도 있다.

## segregated-storage 탐색(find-free-index)

사이즈를 할당하기 위해 segregated-storage을 탐색하는 함수를 구현해야 한다.

```code
static int find_free_index(size_t adjust_size){
    for (int i = 0; i < 30 ; i++){
        if (adjust_size == 1<<(i+3)){
            return i;
        }
    }
}
```
adjust_size 가 어떤 size-class(array index)에 해당하는지 찾는 함수이다. 
```code
static int is_NULL(int index){
    if (segregated_free_list[index] == 0){
        return 1;
    }
    return 0;
}
```
find-free-index 에서 찾은 index를 활용해 segregated_storage에 free-list가 있는지 확인한다.

## page를 일정한 크기로 분할(divide_chunk)
```code
static void* extend_heap(size_t words, size_t adjust_size){
    unsigned int *bp;
    size_t size;
    size = even_number_words_alignment(words);
    bp = mem_sbrk(size);
    if((long)bp == -1){
        return NULL;
    }
    PUT(mem_heap_hi()-3, PACK(0,1));
    divide_chunk(adjust_size, bp);
}
```
```code
static void divide_chunk(size_t adjust_size, void* bp){
    int index = find_free_index(adjust_size);
    segregated_free_list[index] = bp;
    while(1){
        if ((char*)bp + adjust_size > mem_heap_hi() - 3){
            PUT(NEXT(bp), 0);
            break;
        }
        PUT(NEXT(bp), (unsigned int*)((char*)bp + adjust_size));
        bp = GET(NEXT(bp));
    }
}
```

extend_heap()으로 heap size를 확장하고, 시작주소 bp에서부터 마지막 주소까지 순회하며 Next블록을 초기화해준다.

전부 같은 size로 나누기때문에 시작 주소에 adjust_size만큼 더하면 다음 주소값을 구할 수 있다.

## 전체 코드
https://github.com/Recordum/MallocLab_impl/blob/main/mm-simply-segreageted.c



# 참고자료
https://stackoverflow.com/questions/64096093/how-does-simple-segregated-storage-pre-allocate-lists-of-fixed-block-sizes

CSAPP - 9.9.14 

