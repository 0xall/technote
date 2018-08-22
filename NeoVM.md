## Exception 문제

Neo boa compiler 로 `raise Exception(msg)` 를 컴파일하면

```
LOAD_CONST     msg
THROW
```

으로 변환하던데 `0xf0` 은 https://github.com/ontio/neo-boa/blob/master/boa/interop/VMOp.py#L139 에서 보면 알 수 있듯이 Throw 가 맞지만 
Throw 가 발생해도 Halt(스토리지 값 변경 없이, 즉 롤백하고 강제 종료?) 되는게 아니라 계속 다음 코드가 계속 실행된다.

우리는 다음과 같은 contract를 하나 생성하였다

```python
from boa.interop.System.Storage import *

def Main():
    Put(GetContext(), 'test', 1)
    raise Exception()
    Put(GetContext(), 'test', 2)
```

코드를 컴파일하고 나온 OpCode와 Code Hash 는 다음과 같다.

### OpCode

```
55c56b681953797374656d2e53746f726167652e476574436f6e74657874610474657374515272681253797374656d2e53746f726167652e50757461f0681953797374656d2e53746f726167652e476574436f6e74657874610474657374525272681253797374656d2e53746f726167652e50757461006c7566
```

### Debug

```
2             3    CALL_FUNCTION       GetContext()                                      [data] 187160553006994482095689834854674154643729186136794907761267481
              31   LOAD_CONST          test                                              [data] 1953719668          
              36   LOAD_CONST          1                                                 [data]                     
              39   CALL_FUNCTION       Put(a,b,c)                                        [data] 2597105864743928505907994350136493159201985298


3             60   CALL_FUNCTION       Exception()                                       [data]                     


4             61   CALL_FUNCTION       GetContext()                                      [data] 187160553006994482095689834854674154643729186136794907761267481
              89   LOAD_CONST          test                                              [data] 1953719668          
              94   LOAD_CONST          2                                                 [data]                     
              97   CALL_FUNCTION       Put(a,b,c)                                        [data] 2597105864743928505907994350136493159201985298
              121  RETURN_VALUE                                                          [data]                     

```

### Code Hash
```
9be3865b79995a3014be25f541266f3a5d966f60
```
 
중간에 Exception 이 발생했기 때문에 우리는 트랜잭션이 Halt 되어서 'test' 키 값의 value가 아무것도 안들어가 있는(0) 상태로 예상을 했지만 결과값이 2가
나왔다. 즉, Exception 이 발생해도 끝까지 실행된 결과를 보여주었다.


우리는 Contract Code를 작성할 때 Ethereum의 require 처럼 transaction function 이 호출되고 실행되다가 특정 조건을 만족하지 못하면 Transaction
을 어느 값도 변경없이 롤백시키고 종료시키는 함수가 필요하다. 이는 프로젝트의 유지 및 관리 측면과 가독성 면에서 if로 분기시켜가며 조건 체크를 하는 것보다 더
좋을 것이라 생각한다. 

아래는 Neo Boa에 있는 example에서 NEP-5와 관련된 코드를 가져온 것이다. 
(https://github.com/ontio/neo-boa/blob/master/boa_test/example/demo/NEP5.py#L177)

```python
...
    if amount <= 0:
        Log("Cannot transfer negative amount")
        return False

    from_is_sender = CheckWitness(t_from)

    if not from_is_sender:
        Log("Not owner of funds to be transferred")
        return False

    if t_from == t_to:
        Log("Sending funds to self")
        return True

    context = GetContext()

    from_val = Get(context, t_from)

    if from_val < amount:
        Log("Insufficient funds to transfer")
        return False
...
```

우리는 `Require(condition)` 이라는 함수를 만들고 다음과 같이 바꾸었다.
```python
...
    Require(amount > 0)               # cannot transfer minus value
    Require(len(t_to) == 20)          # receiver address validation
    Require(CheckWitness(t_from))     # transaction sender should be sender
    Require(t_from != t_to)           # sender and receiver cannot be the same
...
```

우리는 library 형태로 파일을 따로 만들어서 `Require` 함수는 다음과 같이 작성했는데

```python
def Halt():
    throw Exception(0xFFFFFFFFFFFFFFFF)

def Require(condition):
    if not condition:
        Halt()
    return True
```

컴파일을 한 후에 결과값으로 나온 OpCode에서 Halt 함수의 Op Code인 `09ffffffffffffffff00f0`를 `ffffffffffffffffffffff`로 변환하였다. 
`ff`는 Op Code 상에 존재하지 않아서 아마 함수 실행 전으로 롤백하고 종료되는 것 같은데(아직 우리가 NeoVM Internal에서 어떻게 돌아가는 지 까지는
파악하지 못했다) Exception이 역할을 제대로 수행하지 않아서 우리는 임시로 `0xff` 라는 존재하지 않는 opcode 를 호출해서 함수를 강제로 종료시켰다.

추가적으로 우리는 우리가 만든 `Require` 함수를 응용하여 Safe하게 unsigned int 를 계산하는 함수를 다음과 같이 만들었다.

SafeMath.py
```python
...

def Sub(x, y):
    """
    Calculate x - y. If the result is minus value,
    halt the transaction.
    :param x: x
    :param y: y
    :return: x - y if not minus, else halt the transaction.
    """
    Require(x >= y)
    return x - y

...
```

이를 사용하면 https://github.com/ontio/neo-boa/blob/master/boa_test/example/demo/NEP5.py#L195 와 같이

```python
...
    if from_val < amount:
        Log("Insufficient funds to transfer")
        return False

    if from_val == amount:
        Delete(context, t_from)

    else:
        difference = from_val - amount
        Put(context, t_from, difference)
...
```

보다 더 간결하게

```python
    difference = Sub(from_val, amount)
    if difference == 0:
        Delete(context, t_from)
    else:
        Put(context, t_from, difference)
```

작성할 수 있고, +, - 를 사용하지 않고, Add, Sub 같이 Safe Check 를 하고 Safe 하지 않으면 트랜잭션을 강제 종료, Safe 하면 결과값을 리턴해주는 함수를
만들면 코드가 더 간결해지고, 실수를 하는 일이 줄어들 것이라 생각한다.


Neo dApp의 NEP-5 기반의 토큰과 여러 smart contracts 코드를 참고한 결과, 스토리지 값을 변경하기 전에 `if` 문으로 체크해서 아니면 `False`를 리턴하는 
방식으로 하는 것 같은데 우리 생각에는 `Require` 같은 함수가 있으면 코드 가독성과 유지 보수 측면에서 많은 장점이 될 것 같다고 생각한다. 

우리는 `Require` 함수를 만들었지만 OpCode List 에 없는 값을 호출시켜서 강제 종료 시킨 것이라 좋은 방법이라 생각하지 않는다. `f0` 가 실행되면 `revert`
시키거나 `ff` 등 안쓰는 OpCode를 하나 사용해서 Transaction 을 Revert 시키는 OpCode 를 하나 만들어줬으면 좋겠다.
