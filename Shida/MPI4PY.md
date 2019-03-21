# MPI4PY
## 概述

__MPI__(Message Passing Interface, 消息传递接口)，是一个标准化和轻便的能够运行在各种各样并行计算机上的消息传递系统。
消息传递指的是并行执行的各个进程拥有自己**独立的堆栈和代码段**，作为互不相关的多个程序独立执行。
进程之间的信息交互完全通过显式地调用**通信函数**完成。

mpi4py是构建在MPI上的Python非官方库，使python的数据可以在进程之间传递。

## 执行模型

并行程序是指一组独立、同一的处理过程：
- 所有进程包含相同的代码
- 进程可以在不同的节点或者*不同的计算机*
- 当使用python，使用n个python解释器

> Parallel program -> {Process 1, Process 2, ..., Process N}

`mpirun -np 32 python parallel_script.py`

### 基本概念

__rank__: 给予每个进程id
- 可通过rank查询
- 根据rank执行不同的任务

__conmmunicator__: 包含进程的群组
- mpi4py中基本的对象，通过它来调用方法
- MPI\_COMM\_WORLD,包含所有进程

### 数据模型

所有的变量和数据结构都是进程的*局部值*

进程之间通过发送和接收消息来交换数据

## 安装mpi4py (openmpi)

1. 需要一个MPI的实现软件，可以是OPENMPI或MPICH，我使用OPENMPI。  

	1. 从[官网](//www.open-mpi.org/software/ompi/v1.8/downloads/openmpi-1.8.4.tar.gz)上下载最新版本的安装包  

	2. 在解压包路径下解压并配置，最后安装到/usr/local/openmpi/路径  
		`$ tar -zxvf openmpi-1.8.4.tar.gz`  
		`$ cd openmpi-1.8.4`  
		`$./configure --prefix='/usr/local/openmpi'`  
		
	3. build并安装  
		`$ make`  
		`$ sudo make install`  
		
	4. 添加环境变量  
		`$ sudo vi /etc/profile`  
		在最后添加  
		`export PATH=${PATH}:/usr/local/openmpi/bin`  
		`export LD_LIBRARY=${LD_LIBRARY}:/usr/local/openmpi/lib`  
		执行`$ sudo source /etc/profile`启用环境变量配置  
		
	5. 配置openmpi对应的动态链接库路径  
		`$ sudo vi /etc/ld.so.conf`  
		在最后添加`/usr/local/openmpi/lib`  
		使生效`$ sudo ldconfig`  
		
2. python2.7或python3.3+

3. 安装mpi4py

- 使用pip安装
- 
如果有root权限：  
	`$ pip install mpi4py`  
 如果没有root权限，可以安装mpi4py到自己的$HOME下，这样只能自己使用:  
	`$ pip install mpi4py --user`  
保证安装的可执行文件路径~/.local/bin添加到PATH环境变量中，库文件路径~/.local/lib添加到了LD_LIBRARY_PATH中

- 使用源代码安装
	1. 从[mpi4py](//pypi.python.org/pypi/mpi4py)上下载安装包并解压  
		`$ tar -zxvf mpi4py-x.y.z.tar.gz`  
		`$ cd mpi4py-x.y.z`
		
	2. 编译安装包  
		`$ python setup.py build`  
		
	3. 安装mpi4py  
		`$ python setup.py install`
		
	4. 进入mpi.cfg，修改[openmpi]对应配置  
		`$ vi mpi.cfg`  
		```
		[openmpi]
		mpi_dir					= /usr/local/openmpi/
		mpicc					= %(mpi_dir)s/bin/mpicc
		mpicxx					= %(mpi_dir)s/bin/mpixx
		#include_dirs			= %(mpi_dir)s/include
		#libraries				= mpi
		library_dirs			= %(mpi_dir)s/lib
		runtime_library_dirs	= %(library_dirs)s
		```
		
	5. 测试mpi4py  
		`$ python`  
		`>> from mpi4py import MPI`  
		出现回复则测试通过。
	
	
## 使用mpi4py

#### 点对点通信

- 传递通用python对象（阻塞方式）

简单易用、操作不高效、阻塞进程

```
# p2p_blocking.py

from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

if rank == 0:
	data = {'a':1,'b':3.14}
	print 'process %d sends %s' % (rank,data)
	comm.send(data,dest=1,tag=11)
elif rank == 1:
	data = comm.recv(source=0,tag=11)
	print 'process %d receives %s' % (rank,data)
```

运行结果如下

```
$ mpiexec -n 2 python p2p_blocking.py
process 0 sends {'a':1,'b':3.14}
process 1 receives {'a':1,'b':3.14}
```

- 传递通用python对象（非阻塞方式）

简单易用、不高效、通信计算重叠改善性能

```
# p2p_non_blocking.py

from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

if rank == 0:
	data = {'a':1,'b':3.14}
	print 'process %d sends %s' % (rank,data)
	req = comm.send(data,dest=1,tag=11)
	req.wait()
elif rank == 1:
	data = comm.recv(source=0,tag=11)
	data = req.wait()
	print 'process %d receives %s' % (rank,data)
```

运行结果如下：

```
$ mpiexec -n 2 python p2p_non_blocking.py
process 0 sends {'a':1,'b':3.14}
process 1 receives {'a':1,'b':3.14}
```

- 传递numpy数组（高效快速的方式）

对于数组有单段缓冲区接口(single-segment buffer interface)，不需要pickle序列化，需要使用的通信子对象首字母大写

```
# p2p_numpy_array.py

import numpy
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# passing MPI datatypes explicitly
if rank == 0:
    data = numpy.arange(10, dtype='i')
    print 'process %d sends %s' % (rank, data)
    comm.Send([data, MPI.INT], dest=1, tag=77)
elif rank == 1:
    data = numpy.empty(10, dtype='i')
    comm.Recv([data, MPI.INT], source=0, tag=77)
    print 'process %d receives %s' % (rank, data)

# automatic MPI datatype discovery
if rank == 0:
    data = numpy.arange(10, dtype=numpy.float64)
    print 'process %d sends %s' % (rank, data)
    comm.Send(data, dest=1, tag=13)
elif rank == 1:
    data = numpy.empty(10, dtype=numpy.float64)
    comm.Recv(data, source=0, tag=13)
    print 'process %d receives %s' % (rank, data)
```

运行结果如下

```
$ mpiexec -n 2 python p2p_numpy_array.py
process 0 sends [0 1 2 3 4 5 6 7 8 9]
process 1 receives [0 1 2 3 4 5 6 7 8 9]
process 0 sends [0. 1. 2. 3. 4. 5. 6. 7. 8. 9.]
process 1 receives [0. 1. 2. 3. 4. 5. 6. 7. 8. 9.]
```

#### 集合通信

- 广播(Broadcast)

广播会将根进程的数据复制到同组内的其他所有进程中

广播通用python对象

```
# bcast.py

from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

if rank == 0:
    data = {'key1' : [7, 2.72, 2+3j],
            'key2' : ( 'abc', 'xyz')}
    print 'before broadcasting: process %d has %s' % (rank, data)
else:
    data = None
    print 'before broadcasting: process %d has %s' % (rank, data)

data = comm.bcast(data, root=0)
print 'after broadcasting: process %d has %s' % (rank, data)
```

运行结果如下：

```
$ mpiexec -n 2 python bcast.py
before broadcasting: process 0 has {'key2': ('abc', 'xyz'), 'key1': [7, 2.72, (2+3j)]}
after broadcasting: process 0 has {'key2': ('abc', 'xyz'), 'key1': [7, 2.72, (2+3j)]}
before broadcasting: process 1 has None
after broadcasting: process 1 has {'key2': ('abc', 'xyz'), 'key1': [7, 2.72, (2+3j)]}
```

广播numpy数组

```
# Bcast.py

import numpy as np
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

if rank == 0:
    data = np.arange(10, dtype='i')
    print 'before broadcasting: process %d has %s' % (rank, data)
else:
    data = np.zeros(10, dtype='i')
    print 'before broadcasting: process %d has %s' % (rank, data)

comm.Bcast(data, root=0)

print 'after broadcasting: process %d has %s' % (rank, data)
```

运行结果如下：

```
$ mpiexec -n 2 python Bcast.py
before broadcasting: process 0 has [0 1 2 3 4 5 6 7 8 9]
after broadcasting: process 0 has [0 1 2 3 4 5 6 7 8 9]
before broadcasting: process 1 has [0 0 0 0 0 0 0 0 0 0]
after broadcasting: process 1 has [0 1 2 3 4 5 6 7 8 9]
```

#### 发散(Scatter)

发散操作从组内的根进程分别向组内其他进程散发不同的消息

发散通用的Python对象

```
# scatter.py

from mpi4py import MPI

comm = MPI.COMM_WORLD
size = comm.Get_size()
rank = comm.Get_rank()

if rank == 0:
    data = [ (i + 1)**2 for i in range(size) ]
    print 'before scattering: process %d has %s' % (rank, data)
else:
    data = None
    print 'before scattering: process %d has %s' % (rank, data)

data = comm.scatter(data, root=0)
print 'after scattering: process %d has %s' % (rank, data)
```

运行结果如下

```
$ mpiexec -n 3 python scatter.py
before scattering: process 0 has [1, 4, 9]
after scattering: process 0 has 1
before scattering: process 1 has None
after scattering: process 1 has 4
before scattering: process 2 has None
after scattering: process 2 has 9
```

发散numpy数组

```
# Scatter.py

import numpy as np
from mpi4py import MPI

comm = MPI.COMM_WORLD
size = comm.Get_size()
rank = comm.Get_rank()

sendbuf = None
if rank == 0:
    sendbuf = np.empty([size, 10], dtype='i')
    sendbuf.T[:, :] = range(size)
print 'before scattering: process %d has %s' % (rank, sendbuf)

recvbuf = np.empty(10, dtype='i')
comm.Scatter(sendbuf, recvbuf, root=0)
print 'after scattering: process %d has %s' % (rank, recvbuf)
```

运行结果如下：

```
$ mpiexec -n 3 python Scatter.py
before scattering: process 0 has [[0 0 0 0 0 0 0 0 0 0]
[1 1 1 1 1 1 1 1 1 1]
[2 2 2 2 2 2 2 2 2 2]]
before scattering: process 1 has None
before scattering: process 2 has None
after scattering: process 0 has [0 0 0 0 0 0 0 0 0 0]
after scattering: process 2 has [2 2 2 2 2 2 2 2 2 2]
after scattering: process 1 has [1 1 1 1 1 1 1 1 1 1]
```

#### 收集(Gather)

收集是发散的逆操作，根进程从其他进程收集不同的消息依次放入自己的接收缓冲区内

收集通用python对象

```
# gather.py

from mpi4py import MPI

comm = MPI.COMM_WORLD
size = comm.Get_size()
rank = comm.Get_rank()

data = (rank + 1)**2
print 'before gathering: process %d has %s' % (rank, data)

data = comm.gather(data, root=0)
print 'after scattering: process %d has %s' % (rank, data)
```

运行结果如下

```
$ mpiexec -n 3 python gather.py
before gathering: process 0 has 1
after scattering: process 0 has [1, 4, 9]
before gathering: process 1 has 4
after scattering: process 1 has None
before gathering: process 2 has 9
after scattering: process 2 has None
```

收集numpy数组

```
# Gather.py

import numpy as np
from mpi4py import MPI

comm = MPI.COMM_WORLD
size = comm.Get_size()
rank = comm.Get_rank()

sendbuf = np.zeros(10, dtype='i') + rank
print 'before gathering: process %d has %s' % (rank, sendbuf)

recvbuf = None
if rank == 0:
    recvbuf = np.empty([size, 10], dtype='i')

comm.Gather(sendbuf, recvbuf, root=0)
print 'after gathering: process %d has %s' % (rank, recvbuf)
```

运行结果如下

```
$ mpiexec -n 3 python Gather.py
before gathering: process 0 has [0 0 0 0 0 0 0 0 0 0]
after gathering: process 0 has [[0 0 0 0 0 0 0 0 0 0]
[1 1 1 1 1 1 1 1 1 1]
[2 2 2 2 2 2 2 2 2 2]]
before gathering: process 1 has [1 1 1 1 1 1 1 1 1 1]
after gathering: process 1 has None
before gathering: process 2 has [2 2 2 2 2 2 2 2 2 2]
after gathering: process 2 has None
```

#### 比较send()/recv()

```
# send_recv_timing.pu

import time
import numpy as np
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

if rank == 0:
    data = np.random.randn(10000).astype(np.float64)
else:
    data = np.empty(10000, dtype=np.float64)

comm.barrier()

# use comm.send() and comm.recv()
t1 = time.time()
if rank == 0:
    comm.send(data, dest=1, tag=1)
else:
    comm.recv(source=0, tag=1)
t2 = time.time()
if rank == 0:
    print 'time used by send/recv: %f seconds' % (t2 - t1)

comm.barrier()

# use comm.Send() and comm.Recv()
t1 = time.time()
if rank == 0:
    comm.Send(data, dest=1, tag=2)
else:
    comm.Recv(data, source=0, tag=2)
t2 = time.time()
if rank == 0:
    print 'time used by Send/Recv: %f seconds' % (t2 - t1)
```

运行结果如下

```
$ mpiexec -n 2 python send_recv_timing.py
time used by send/recv: 0.000412 seconds
time used by Send/Recv: 0.000091 seconds
```

可以看出在代码几乎一样的情况下，以大写字母开头的Send()/Recv()方法对numpy数组的传递效率要高的多，
因此在涉及 numpy 数组的并行操作时，应尽量选择以__大写字母__开头的通信方法。

## 总结
以上通过几个简单的例子介绍了怎么在Python中利用mpi4py进行并行编程，可以看出mpi4py使得在Python中进行MPI并行编程非常容易，
也比在C、C++、Fortran中调用MPI的应用接口进行并行编程要__方便和灵活__的多，
特别是mpi4py提供的基于pickle的通用Python对象传递机制，使我们在编程过程中完全不用考虑所传递的数据类型和数据长度。
这种灵活性和易用性虽然会有一些性能上的损失，但是在传递的数据量不大的情况下，这种__性能损失是可以忽略的__。

当需要传递大量的数组类型的数据时，mpi4py提供的以大写字母开头的通信方法使得数据可以以__接近__C、C++、Fortran的速度在不同的进程间高效地传递。
对numpy数组，这种高效性却并不损失或很少损失其灵活性和易用性，因为mpi4py可以__自动推断__出numpy数组的类型及数据长度信息，
因此一般情况下不用显式的指定。这给我们利用numpy的数组进行高性能的并行计算编程带来莫大的方便。


> *参考*  
> 1 https://www.jianshu.com/p/f497f3a5855f  
> 2 https://stackoverflow.com/questions/4581305/error-while-loading-shared-libraries-libboost-system-so-1-45-0-connot-open-sha  
> 3 https://www.cnblogs.com/simba372/p/5311657.html  
> 4 https://www.cnblogs.com/zhbzz2007/p/5827059.html