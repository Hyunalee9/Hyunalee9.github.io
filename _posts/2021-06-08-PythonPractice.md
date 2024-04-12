---
title: "[백준] 1110번 더하기 사이클"
excerpt: "코딩 , 알고리즘 공부"
categories:
   - Python
tags:
   - TIL
   - Python
last_modified_at : 2021-06-08 T20:17
---
__문제__

0보다 크거나 같고, 99보다 작거나 같은 정수가 주어질 때 다음과 같은 연산을 할 수 있다. 먼저 주어진 수가 10보다 작다면 앞에 0을 붙여 두 자리 수로 만들고, 각 자리의 숫자를 더한다. 그 다음, 주어진 수의 가장 오른쪽 자리 수와 앞에서 구한 합의 가장 오른쪽 자리 수를 이어 붙이면 새로운 수를 만들 수 있다. 다음 예를 보자.

26부터 시작한다. 2+6 = 8이다. 새로운 수는 68이다. 6+8 = 14이다. 새로운 수는 84이다. 8+4 = 12이다. 새로운 수는 42이다. 4+2 = 6이다. 새로운 수는 26이다.

위의 예는 4번만에 원래 수로 돌아올 수 있다. 따라서 26의 사이클의 길이는 4이다.

N이 주어졌을 때, N의 사이클의 길이를 구하는 프로그램을 작성하시오.

---

__접근 방법__

n을 리스트에 넣어 준 후 두번째 값 원소와 세번째 값 원소를
이용했다.

이를 가지고 새롭게 생성된 new_number 가 입력받은 값과 같을 때까지 while문을 돌려줬다. 입력 받은 값이 한 자리 수인 경우 등등 예외처리도 해줬다.

__내 코드__

```python
n = input()
new_number = ""
cnt = 0
n_array =[int(_) for _ in n] # n을 쪼개 int로 변환 후 n_array에 넣어준다.
if len(n_array) < 2:
    n_array.append(n_array[0])
    n_array[0] = 0  # ex) [0,1] , [0,0]

n_array.append(n_array[0] + n_array[1])  # n_array = [2, 6, 8]

while new_number != n:
    cnt += 1
    new_number = str(n_array[1]) + str(n_array[2])  # 68 , 84, 42
    if n_array[1] == 0:       # 01, 02 같은 것들 처리..
       new_number = str(n_array[2])
    add_number = str(n_array[1] + n_array[2])  # 14 , 12, 6 , 26
    n_array[1] = n_array[2]
    if len(add_number) == 1:  # ex) 한 자리 수일경우
        n_array[2] = int(add_number[0])
    else:
        n_array[2] = int(add_number[1])  
print(cnt)
```
