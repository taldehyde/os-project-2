# Nachos Project2 Report 
## b99901131 卓旻科

### 1. Implement Sleep(int) system call


Define新的system call code並且宣告`void Sleep(int x)`

in userprog/syscall.h

    #define SC_Sleep	12
    ...
    void Sleep(int x);		//sleep system call 

增加新的exeption來處理SC_Sleep system call

in userprog/exeption.cc

	switch(type){
		...
    	case SC_Sleep:
			val=kernel->machine->ReadRegister(4);
			kernel->alarm->WaitUntil(val);
			return;
		...
	}

在`Alarm::WaitUntil(int x)`裡處理sleep system call

in threads/alarm.cc

    void Alarm::WaitUntil(int x)
    {
    	kernel->scheduler->SetSleeping(x);
    }

將正在進行的thread讓他暫時sleeping，並使用`sleepingList`來記錄sleeping threads。
而`sleepingList`是一個SortedList，剩餘sleeping時間最小的會排在最前面。

in threads/scheduler.cc

	void 
	Scheduler::SetSleeping(int sleepTime)
	{
    	kernel->interrupt->SetLevel(IntOff);
    	Thread* thread = kernel->currentThread;
		...
		sleepingList->Insert(SleepingThread(thread,sleepTime));
    	thread->Sleep(FALSE);
    	kernel->interrupt->SetLevel(IntOn);
	}

Alarm的設定是每100 ticks會產生interrupt，並呼叫`Alarm::CallBack()`，在這裡呼叫
`kernel->scheduler->AlarmTicks()`，以提醒scheduler將剩餘睡眠次數減一，並將睡眠次數歸零的thread重新排入readyList。

in threads/alarm.cc

	void 
	Alarm::CallBack() 
	{
	    //manage sleeping threads
	    bool slEmpty = kernel->scheduler->IsSleepingListEmpty();
    	kernel->scheduler->AlarmTicks();
	
    	Interrupt *interrupt = kernel->interrupt;
    	MachineStatus status = interrupt->getStatus();
    	if (status == IdleMode && slEmpty) {	// is it time to quit?
    	    if (!interrupt->AnyFutureInterrupts()) {
		    	timer->Disable();	// turn off the timer
			}
    	} else {			// there's someone to preempt
    	    if(kernel->scheduler->getSchedulerType() == SJF){
    	        return;
    	    }
			interrupt->YieldOnReturn();
    	}
	}

in threads/scheduler.cc

	void 
	Scheduler::AlarmTicks()
	{
    	ListIterator<SleepingThread> iter(sleepingList);
    	for (; !iter.IsDone(); iter.Next()) {
    	    iter.Item().decreaseTime(1);
    	}	
    	while(!sleepingList->IsEmpty()){
    	    SleepingThread st = sleepingList->Front();
    	    if(st.getSleepTime() == 0){
    	        ReadyToRun(st.getThread());
    	        sleepingList->RemoveFront();
    	    }
    	    else { break; }
    	}
	}

### 2. Implement Non-preemptive shortest job first scheduler

在kernel裡增加指令，讓我們可以用`./nachos -SJF ...`來使用SJF。

in threads/kernel.cc

	ThreadedKernel::ThreadedKernel(int argc, char **argv)
	{
    	randomSlice = FALSE; 
    	sType = RR;
    	for (int i = 1; i < argc; i++) {
    	    if (strcmp(argv[i], "-rs") == 0) {
 		    ASSERT(i + 1 < argc);
		    RandomInit(atoi(argv[i + 1]));// initialize pseudo-random
						// number generator
		    randomSlice = TRUE;
		    i++;
    	    } 
    	    else if (strcmp(argv[i], "-u") == 0) {
    	        cout << "Partial usage: nachos [-rs randomSeed]\n";
    	        cout << "Partial usage: nachos [-SJF]\n";
			}
    		else if (strcmp(argv[i], "-SJF") == 0) {
    	       sType = SJF;
    	    }
    	}
	}

	void
	ThreadedKernel::Initialize()
	{
    	stats = new Statistics();		// collect statistics
    	interrupt = new Interrupt;		// start up interrupt handling
    	scheduler = new Scheduler(sType);	// initialize the ready queue
    	alarm = new Alarm(randomSlice);	// start up time slicing
		...
	}

我們想讓thread在sleep的時候再交出CPU的使用權，而預設的RR切換的時間點在`interrupt->YieldOnReturn();`這裡。使用SJF時我們不希望有切換，所以直接`return;`

in threads/alarm.cc

	void 
	Alarm::CallBack() 
	{
	    //manage sleeping threads
	    bool slEmpty = kernel->scheduler->IsSleepingListEmpty();
    	kernel->scheduler->AlarmTicks();
	
    	Interrupt *interrupt = kernel->interrupt;
    	MachineStatus status = interrupt->getStatus();
    	if (status == IdleMode && slEmpty) {	// is it time to quit?
    	    if (!interrupt->AnyFutureInterrupts()) {
		    	timer->Disable();	// turn off the timer
			}
    	} else {			// there's someone to preempt
    	    if(kernel->scheduler->getSchedulerType() == SJF){
    	        return;
    	    }
			interrupt->YieldOnReturn();
    	}
	}

當thread要sleeping的時候，我們紀錄它實際使用CPU的時間，在這裡等同於`kernel->stats->userTicks`，因為執行一個user instruction，`userTicks`就會+1。
因此我們紀錄thread開始到到sleep間的`userTicks`變化，來得到實際的CPU burst，並配合預測公式來得到我們預測的next CPU burst。
我們將預測出來的值用`burstTimeMap`(`std::map`)來管理，如此一來查詢時會比較方便。

in threads/scheduler.cc

	void 
	Scheduler::SetSleeping(int sleepTime)
	{
	    kernel->interrupt->SetLevel(IntOff);
	    Thread* thread = kernel->currentThread;

	    if(schedulerType == SJF){
	        //record cpu burst(user ticks)
	        ASSERT(burstTimeMap->find(thread) != burstTimeMap->end());
	
	        int prevBurst = (*burstTimeMap)[thread];
	        int actualBurst = kernel->stats->userTicks - prevTicks;
	        int estiBurst = (int) (actualBurst*RATE + prevBurst*(1-RATE));
	        (*burstTimeMap)[thread] = estiBurst;
	        prevTicks = kernel->stats->userTicks;
	        cout << "\nThread " << thread->getName() 
	             << "\nprevBurst:\t" << prevBurst
	             << "\nactualBurst:\t" << actualBurst
	             << "\nestiBurst:\t"   << estiBurst << endl << endl;
	    }

	    sleepingList->Insert(SleepingThread(thread,sleepTime));
	    thread->Sleep(FALSE);
	    kernel->interrupt->SetLevel(IntOn);
	}

當使用SJF時，我們必須將原本的`readyList`改成SortedList，並使用預測出的CPU burst來排`readyList`。
而第一次加入readyList的thread，我們將其預測的CPU burst設為0。

in threads/scheduler.cc

	int burstCompare(Thread* a, Thread* b)
	{
	    int aTime = (*(kernel->scheduler->burstTimeMap))[a];
	    int bTime = (*(kernel->scheduler->burstTimeMap))[b];
	    if(aTime == bTime) { return 0; }
	    if(aTime > bTime) { return 1; }
	    if(aTime < bTime) { return -1; }
	}

	Scheduler::Scheduler(SchedulerType type)
	{
		schedulerType = type;
		readyList = new List<Thread *>; 
		sleepingList = new SortedList<SleepingThread> (mySTCompare);
	    if(schedulerType == SJF){
	        delete readyList;
	    	burstTimeMap = new map<Thread* , int>;
	    	readyList = new SortedList<Thread *>(burstCompare); 
		}
		toBeDestroyed = NULL;
	} 

	void
	Scheduler::ReadyToRun (Thread *thread)
	{
	    ASSERT(kernel->interrupt->getLevel() == IntOff);
	    DEBUG(dbgThread, "Putting thread on ready list: " << thread->getName());
	    if(schedulerType == SJF){     
	        if(burstTimeMap->find(thread) == burstTimeMap->end()){
	            //thread not found
	            (*burstTimeMap)[thread] = 0;
	        }
	    }	    
	    thread->setStatus(READY);
	    readyList->Append(thread);
	}

###Result

使用了以下三個檔案來做測試。
我們可以從第一個數字來辨別每個output分別是屬於哪一個thread。

test/sjf_test1.c

	int
	main()
	{
	    int i;
	    int j;
	    for(i = 0; i < 9; i++){
	        for(j = 0; j < 5; j++){
	            PrintInt(100+i*10+j);
	        }
	        Sleep(1);
	    }
	}

test/sjf_test2.c

	int
	main()
	{
	    int i;
	    int j;
	    for(i = 0; i < 9; i++){
	        for(j = 0; j < 9; j++){
	            PrintInt(200+i*10+j);
	        }
	        Sleep(1);
	    }
	}

test/sjf_test3.c

	int
	main()
	{
	    int i;
	    for(i = 0; i < 9; i++){
	        PrintInt(i*10+300);
	        Sleep(1);
	    }
	}

使用預設的RR 
`./userprog/nachos -e ./test/sjf_test1.c -e ./test/sjf_test2.c -e ./test/sjf_test3.c`

結果

`project2_rr`

    Total threads number is 3
    Thread ./test/sjf_test1 is executing.
    Thread ./test/sjf_test2 is executing.
    Thread ./test/sjf_test3 is executing.
    Print integer:100
    Print integer:200
    Print integer:201
    Print integer:202
    Print integer:300
    Print integer:101
    Print integer:102
    Print integer:203
    Print integer:204
    Print integer:205
    Print integer:310
    Print integer:103
    Print integer:104
    Print integer:206
    Print integer:207
    Print integer:208
    Print integer:210
    Print integer:211
    Print integer:320
    Print integer:110
    Print integer:212
    Print integer:213
    Print integer:214
    Print integer:215
    Print integer:330
    Print integer:111
    Print integer:112
    Print integer:216
    Print integer:217
    Print integer:218
    Print integer:340
    Print integer:113
    Print integer:114
    Print integer:350
    Print integer:360
    Print integer:120
    Print integer:220
    Print integer:221
    Print integer:222
    Print integer:370
    Print integer:121
    Print integer:122
    Print integer:223
    Print integer:224
    Print integer:225
    Print integer:380
    Print integer:123
    Print integer:124
    Print integer:226
    Print integer:227
    Print integer:228
    Print integer:230
    Print integer:231
    return value:0
    Print integer:130
    Print integer:131
    Print integer:232
    Print integer:233
    Print integer:234
    Print integer:132
    Print integer:133
    Print integer:134
    Print integer:235
    Print integer:236
    Print integer:237
    Print integer:238
    Print integer:240
    Print integer:241
    Print integer:140
    Print integer:141
    Print integer:142
    Print integer:242
    Print integer:243
    Print integer:244
    Print integer:245
    Print integer:143
    Print integer:144
    Print integer:150
    Print integer:151
    Print integer:152
    Print integer:246
    Print integer:247
    Print integer:248
    Print integer:250
    Print integer:251
    Print integer:252
    Print integer:153
    Print integer:154
    Print integer:253
    Print integer:160
    Print integer:161
    Print integer:162
    Print integer:254
    Print integer:255
    Print integer:256
    Print integer:163
    Print integer:164
    Print integer:257
    Print integer:170
    Print integer:171
    Print integer:172
    Print integer:258
    Print integer:173
    Print integer:260
    Print integer:261
    Print integer:262
    Print integer:174
    Print integer:263
    Print integer:264
    Print integer:180
    Print integer:181
    Print integer:182
    Print integer:265
    Print integer:266
    Print integer:267
    Print integer:183
    Print integer:184
    Print integer:268
    return value:0
    Print integer:270
    Print integer:271
    Print integer:272
    Print integer:273
    Print integer:274
    Print integer:275
    Print integer:276
    Print integer:277
    Print integer:278
    Print integer:280
    Print integer:281
    Print integer:282
    Print integer:283
    Print integer:284
    Print integer:285
    Print integer:286
    Print integer:287
    Print integer:288
    return value:0
    No threads ready or runnable, and no pending interrupts.
    Assuming the program completed.
    Machine halting!
    
    Ticks: total 5100, idle 216, system 840, user 4044
    Disk I/O: reads 0, writes 0
    Console I/O: reads 0, writes 0
    Paging: faults 0
    Network I/O: packets received 0, sent 0
    
使用SJF
`./userprog/nachos -SJF -e ./test/sjf_test1.c -e ./test/sjf_test2.c -e ./test/sjf_test3.c`

結果

`project2_sjf`

    Total threads number is 3
    Thread ./test/sjf_test1 is executing.
    Thread ./test/sjf_test2 is executing.
    Thread ./test/sjf_test3 is executing.
    Print integer:100
    Print integer:101
    Print integer:102
    Print integer:103
    Print integer:104
    
    Thread ./test/sjf_test1
    prevBurst:	0
    actualBurst:	156
    estiBurst:	78
    
    Print integer:200
    Print integer:201
    Print integer:202
    Print integer:203
    Print integer:204
    Print integer:205
    Print integer:206
    Print integer:207
    Print integer:208
    
    Thread ./test/sjf_test2
    prevBurst:	0
    actualBurst:	260
    estiBurst:	130
    
    Print integer:300
    
    Thread ./test/sjf_test3
    prevBurst:	0
    actualBurst:	31
    estiBurst:	15
    
    Print integer:110
    Print integer:111
    Print integer:112
    Print integer:113
    Print integer:114
    
    Thread ./test/sjf_test1
    prevBurst:	78
    actualBurst:	154
    estiBurst:	116
    
    Print integer:310
    
    Thread ./test/sjf_test3
    prevBurst:	15
    actualBurst:	29
    estiBurst:	22
    
    Print integer:120
    Print integer:121
    Print integer:122
    Print integer:123
    Print integer:124
    
    Thread ./test/sjf_test1
    prevBurst:	116
    actualBurst:	154
    estiBurst:	135
    
    Print integer:320
    
    Thread ./test/sjf_test3
    prevBurst:	22
    actualBurst:	29
    estiBurst:	25
    
    Print integer:210
    Print integer:211
    Print integer:212
    Print integer:213
    Print integer:214
    Print integer:215
    Print integer:216
    Print integer:217
    Print integer:218
    
    Thread ./test/sjf_test2
    prevBurst:	130
    actualBurst:	258
    estiBurst:	194
    
    Print integer:330
    
    Thread ./test/sjf_test3
    prevBurst:	25
    actualBurst:	29
    estiBurst:	27
    
    Print integer:130
    Print integer:131
    Print integer:132
    Print integer:133
    Print integer:134
    
    Thread ./test/sjf_test1
    prevBurst:	135
    actualBurst:	154
    estiBurst:	144
    
    Print integer:340
    
    Thread ./test/sjf_test3
    prevBurst:	27
    actualBurst:	29
    estiBurst:	28
    
    Print integer:140
    Print integer:141
    Print integer:142
    Print integer:143
    Print integer:144
    
    Thread ./test/sjf_test1
    prevBurst:	144
    actualBurst:	154
    estiBurst:	149
    
    Print integer:350
    
    Thread ./test/sjf_test3
    prevBurst:	28
    actualBurst:	29
    estiBurst:	28
    
    Print integer:150
    Print integer:151
    Print integer:152
    Print integer:153
    Print integer:154
    
    Thread ./test/sjf_test1
    prevBurst:	149
    actualBurst:	154
    estiBurst:	151
    
    Print integer:360
    
    Thread ./test/sjf_test3
    prevBurst:	28
    actualBurst:	29
    estiBurst:	28
    
    Print integer:160
    Print integer:161
    Print integer:162
    Print integer:163
    Print integer:164
    
    Thread ./test/sjf_test1
    prevBurst:	151
    actualBurst:	154
    estiBurst:	152
    
    Print integer:370
    
    Thread ./test/sjf_test3
    prevBurst:	28
    actualBurst:	29
    estiBurst:	28
    
    Print integer:170
    Print integer:171
    Print integer:172
    Print integer:173
    Print integer:174
    
    Thread ./test/sjf_test1
    prevBurst:	152
    actualBurst:	154
    estiBurst:	153
    
    Print integer:380
    
    Thread ./test/sjf_test3
    prevBurst:	28
    actualBurst:	29
    estiBurst:	28
    
    Print integer:180
    Print integer:181
    Print integer:182
    Print integer:183
    Print integer:184
    
    Thread ./test/sjf_test1
    prevBurst:	153
    actualBurst:	154
    estiBurst:	153
    
    return value:0
    return value:0
    Print integer:220
    Print integer:221
    Print integer:222
    Print integer:223
    Print integer:224
    Print integer:225
    Print integer:226
    Print integer:227
    Print integer:228
    
    Thread ./test/sjf_test2
    prevBurst:	194
    actualBurst:	258
    estiBurst:	226
    
    Print integer:230
    Print integer:231
    Print integer:232
    Print integer:233
    Print integer:234
    Print integer:235
    Print integer:236
    Print integer:237
    Print integer:238
    
    Thread ./test/sjf_test2
    prevBurst:	226
    actualBurst:	258
    estiBurst:	242
    
    Print integer:240
    Print integer:241
    Print integer:242
    Print integer:243
    Print integer:244
    Print integer:245
    Print integer:246
    Print integer:247
    Print integer:248
    
    Thread ./test/sjf_test2
    prevBurst:	242
    actualBurst:	258
    estiBurst:	250
    
    Print integer:250
    Print integer:251
    Print integer:252
    Print integer:253
    Print integer:254
    Print integer:255
    Print integer:256
    Print integer:257
    Print integer:258
    
    Thread ./test/sjf_test2
    prevBurst:	250
    actualBurst:	258
    estiBurst:	254
    
    Print integer:260
    Print integer:261
    Print integer:262
    Print integer:263
    Print integer:264
    Print integer:265
    Print integer:266
    Print integer:267
    Print integer:268
    
    Thread ./test/sjf_test2
    prevBurst:	254
    actualBurst:	258
    estiBurst:	256
    
    Print integer:270
    Print integer:271
    Print integer:272
    Print integer:273
    Print integer:274
    Print integer:275
    Print integer:276
    Print integer:277
    Print integer:278
    
    Thread ./test/sjf_test2
    prevBurst:	256
    actualBurst:	258
    estiBurst:	257
    
    Print integer:280
    Print integer:281
    Print integer:282
    Print integer:283
    Print integer:284
    Print integer:285
    Print integer:286
    Print integer:287
    Print integer:288
    
    Thread ./test/sjf_test2
    prevBurst:	257
    actualBurst:	258
    estiBurst:	257
    
    return value:0
    No threads ready or runnable, and no pending interrupts.
    Assuming the program completed.
    Machine halting!
    
    Ticks: total 4719, idle 335, system 340, user 4044
    Disk I/O: reads 0, writes 0
    Console I/O: reads 0, writes 0
    Paging: faults 0
    Network I/O: packets received 0, sent 0

可以看到執行較少次的`sjf_test3`確實較快完成，而執行最多次的`sjf_test2`也是最後一個完成，
因此這個SJF schedule可以得到證實。
且因為是non-preemptive的方式，所以content switch的次數(相當於system ticks)較少。