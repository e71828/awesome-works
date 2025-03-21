## 最严重的问题

`PAR_Retain` 变量移位4个字节（第1-57被发现移位）；可能是因为 `rope_length_theory_standardization_sub:REAL;(*过往补偿值总和*)`的retain 变量。

与此同时，放弃写入参数使用的 retain 变量；不再使用`PAR_Retain[98].1`。

```pascal
INIT111: BOOL := FALSE; // (烧录后自动写入，即使重新上电，因参数非零，不会覆盖变量)
```

## 版本一 主要点

- 添加 persistent 变量

```pascal
(*20241101 回转速度标定 by 唐金峰*)
VAR RETAIN PERSISTENT
	INIT111: BOOL := TRUE; // (烧录后，立即执行，且终生只运行一次)
END_VAR
```

**失败**，原因是没有建立 persistent 变量列表。

## 版本二 主要点

- 检测是否执行，24个参数的初始化输入。

```pascal
COUNT_STEP:=255;
```

**失败**

## 版本三 主要点

```pascal
VAR RETAIN
	INIT111: BOOL := TRUE; // (烧录后，立即执行，且终生只运行一次)
END_VAR
```

**失败**，新增 `retain` 变量需要在程序中 `Reset Cold`，此举影响其它 retain 变量。**一般做法为 write 值**。

## 版本四 成功

```pascal
VAR
	INIT111: BOOL := TRUE; // (烧录后，读取PAR_Retain，立即执行，且终生只运行一次)
END_VAR
```

```pascal
	INIT111:=PAR_Retain[98].1; // (*20241101 回转速度标定 by 唐金峰*)
```

```pascal
	PAR_Retain[98].1:=INIT111; // (*20241101 回转速度标定 by 唐金峰*)
```

- 后续测试，先在显示器将新增的某个参数置于0，断电后重启，观察参数是否恢复，如果恢复则需要将新增参数代码写到从 retain 指定数组的读取后。

**声明即占用**

```pascal
VAR_GLOBAL RETAIN
//RETAIN最大存储区域1KB
	PAR_Retain : ARRAY[1..1000] OF BYTE;
END_VAR
```

**自动写入24个参数成功**

```pascal
(*自动写参数20230314*)(*20241101 回转速度标定 by 唐金峰*)
array_max_length:=964;(*array_max_length的大小自己根据默认参数数组长度修改,如数组0-1000就填1000*)
IF INIT111 THEN
	NEW_Default_Par_AUTOWRITE_Enable:=TRUE;
	COUNT_STEP:=1;
	conut_auto_write:=941;
	INIT111:=FALSE;
END_IF
IF NEW_Default_Par_AUTOWRITE_Enable THEN
	COUNT_STEP:=COUNT_STEP+1;
	IF COUNT_STEP > 7 THEN (*第7个幸运周期*)
		conut_onecycle := 1;
		WHILE conut_auto_write<=array_max_length AND conut_onecycle<=25 DO
			IF array_default_par_set[conut_auto_write]>0 AND (array_par_read[conut_auto_write]=0 OR array_par_read[conut_auto_write]=65535)
				(*部分参数可能默认不是0，但实际调整至0,如功能配置参数、最低角度参数，需筛选掉*)
			AND (conut_auto_write<>220 AND  conut_auto_write<>237 AND conut_auto_write<>238 AND conut_auto_write<>239 AND conut_auto_write<>921) THEN(*新增的参数*)
				auto_write_value:=array_default_par_set[conut_auto_write];
				F_framwrite_4(ENABLE:=TRUE , DST:=conut_auto_write*2 , LEN:=2 , SRC:=ADR(auto_write_value) );  //更新内存变量
			ELSE(*代表不是新增的*)
				auto_write_value:=0;
			END_IF
			conut_auto_write:=conut_auto_write+1;
			conut_onecycle:=conut_onecycle+1;
		END_WHILE
	END_IF
	IF conut_auto_write>array_max_length THEN(*写完*)
		NEW_Default_Par_AUTOWRITE_Enable:=FALSE;
		enable_par_read:=TRUE; // 下个周期读取内存，更新变量
		COUNT_STEP:=255;
	END_IF
END_IF // (*20241101 回转速度标定 by 唐金峰*)
```

