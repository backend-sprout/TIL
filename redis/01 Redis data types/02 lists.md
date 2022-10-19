# lists 
       
Redis 목록은 문자열 값의 연결 목록입니다.         
       
* 스택과 큐를 구현합니다.         
* 백그라운드 작업자 시스템에 대한 대기열 관리를 구축합니다.          
    
# 예       
**목록을 대기열처럼 취급하십시오(선입 선출)**         
```
> LPUSH work:queue:ids 101   
(integer) 1
> LPUSH work:queue:ids 237
(integer) 2   
> RPOP work:queue:ids     
"101"
> RPOP work:queue:ids
"237"
```

Redis 목록은 다음과 같은 용도로 자주 사용됩니다.                

* LPUSH : 왼쪽으로부터 리스트에 데이터를 추가한다.  
* RPOP : 오른쪽으로부터 리스트에 데이터를 꺼내온다.  
   
**목록을 스택처럼 취급합니다(선입 선출)**     
```
> LPUSH work:queue:ids 101
(integer) 1
> LPUSH work:queue:ids 237
(integer) 2
> LPOP work:queue:ids
"237"
> LPOP work:queue:ids
"101"
```

* LPUSH : 왼쪽으로부터 리스트에 데이터를 추가한다.  
* RPOP : 왼쪽으로부터 리스트에 데이터를 꺼내온다.  
   
**목록을 스택처럼 취급합니다(선입 선출)**.    
```
> LLEN work:queue:ids
(integer) 0
```

* LLEN : 리스트의 길이를 확인한다.   

**한 목록에서 요소를 원자적으로 팝하고 다른 목록으로 푸시합니다.**   

```
> LPUSH board:todo:ids 101
(integer) 1
> LPUSH board:todo:ids 273
(integer) 2
> LMOVE board:todo:ids board:in-progress:ids LEFT LEFT
"273"
> LRANGE board:todo:ids 0 -1
1) "101"
> LRANGE board:in-progress:ids 0 -1
1) "273"
```
* LMOVE : 
* LRANGE : 왼쪽부터 범위를 지정한 만큼의 데이터를 가져온다.   






