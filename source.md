## deep into sourcecode
### yield
```
YIELD_VALUE return and suspends (state) for the coroutine.
```
### robo control
```
connect IP@/Port with TCP
do command by send command
client <----> server mode, server on the fly
```
### abcMeta
```
abstract method decorated, in order that this class can't be instantiated.
but in sub-class where to define the method defination. and it can be defined for the detail.
So this is from language point of view, it limit the abstract class's method to be initialized,
but keep the empty meanful declaration. it's sub-class's turn to define the implementation.
__metaclass__ = ABCMeta

@abc.abstract
@abstractmethod
def xxx(self):
    return 1
```
