# **为OSRFX2创建并发布一个基于WDF的核心驱动程序Step by Step**

# Step1 - 创建一个最简单的KMDF功能驱动。

它只注册了**DriverEntry**和**EvtDeviceAdd**两个回调函数。WDF框架提供了其他所有缺省的功能，使之可以支持加载驱动并响应PNP和电源事件。我们可以安装，卸载，使能，去能该驱动，同时在OS挂起和恢复情况下该最小驱动也可以正常工作。

## *驱动入口点[DriverEntry]*

当一个驱动被加载到内存后第一个被FrameWork调用的函数，我们可以在这个函数中执行驱动的初始化工作。
对标准驱动来说，这个函数是非常简单的：FrameWork在调用这个函数时传入两个参数，参数一DriverObject指向一块由内核的IO Manager分配的内存对象，我们需要在DriverEntry中对它进行初始化。参数二RegistryPath给出了内核在系统注册表中为我们的驱动分配的单元的路径，我们可以在这个路径下去存储一些驱动相关的信息。这里我们没有使用第二个参数。

在初始化DriverObject时，我们必须要用到下面几个重要的数据结构和函数：

- `WDF_DRIVER_CONFIG`和`WDF_DRIVER_CONFIG_INIT`：用于初始化[WdfDriverCreate]的一个入参DriverConfig。`WDF_DRIVER_CONFIG`结构中包含有两个回调函数，我们这里用到了其中一个[EvtDriverDeviceAdd]。每个基于WDF实现的并支持即插即用的驱动都必须实现该回调函数并确保设置好`WDF_DRIVER_CONFIG`中该回调函数的地址，这样在设备连接并被系统枚举时该回调函数才会被系统正确调用到。
- [WdfDriverCreate]：为正在被系统加载的驱动创建一个驱动对象，我们不需要自己申请内存，因为正如前面所述，驱动对象实体已经被FrameWork创建好并给出了其指针句柄。我们的任务就是根据我们的需要对其初始化（虽然函数的名字叫create）。

## *添加新设备入口点[EvtDriverDeviceAdd]*

每当PnP Manager检测到一个新设备，系统都会调用EvtDeviceAdd函数。我们要做的工作就是在此函数中创建并初始化设备对象。

FrameWorks在调用该回调函数时传入两个参数，参数一是驱动对象的句柄，这个应该就是我们在DriverEntry里创建的驱动对象。另一个参数是指向WDFDEVICE_INIT的指针句柄DeviceInit，该句柄是由FrameWork创建的，如果我们不需要对其进行修改，直接传递给[WdfDeviceCreate]

在创建和初始化设备对象时我们用到了一个重要的WDF API [WdfDeviceCreate]。调用WdfDeviceCreate会创建一个框架设备对象(framework device object)来表达一个功能设备对象(functional device object, 简称FDO)或者是一个物理设备对象(physical device object, 简称PDO)。我们这里创建的OSRFX2设备是一个FDO.

在调用WdfDeviceCreate之前，我们根据需要初始化WDFDEVICE_INIT结构对象DeviceInit。在Step1这里我们接受缺省对象，所以什么都不用做。  
Step1中创建设备对象时我们也没有设置对象属性 - `WDF_NO_OBJECT_ATTRIBUTES`, 在Step2中我们可以看到对属性更多的设置。  
`status = WdfDeviceCreate(&DeviceInit, WDF_NO_OBJECT_ATTRIBUTES, &device);`

# Step2 - 准备设备对象
主要包括:

- 为OSRFX2添加上下文
- 为OSRFX2准备硬件IO
- 为OSRFX2注册设备接口([device interface])

## *为OSRFX2添加上下文*
因为我们是基于WDF框架创建的驱动，我们可以为我们驱动的设备对象创建自己的上下文空间。我们可以简单地将上下文空间理解为一块不可分页的内存区域。我们可以将自己驱动程序需要的一些变量定义在这个上下文数据区中，创建好之后我们可以通知WDF将其附加到WDF管理的框架对象中，以便在框架对象的生命周期中访问。更详细的内容我们可以参考[Framework Object Context Space].
这里以OSRFX2为例我们看一下为设备对象创建上下文的过程。

首先我们要自定义一个数据结构DEVICE_CONTEXT.  
`typedef struct _DEVICE_CONTEXT {`  
`  WDFUSBDEVICE      UsbDevice;`  
`  WDFUSBINTERFACE   UsbInterface;`  
`} DEVICE_CONTEXT, *PDEVICE_CONTEXT;`  
在设备上下文结构中我们定义了两个面向底层硬件的WDF对象。具体这两个对象是如何创建的请参考“为OSRFX2准备硬件IO”。  
结构定义好后我们还要调用一个WDF的宏`WDF_DECLARE_CONTEXT_TYPE_WITH_NAME(DEVICE_CONTEXT, GetDeviceContext)`,第一个参数是上面我们定义的上下文数据结构的名字，第二个参数是给出我们期望访问上下文数据结构的函数名称GetDeviceContext。这样定义后，在代码里我们就可以调用GetDeviceContext来得到指向上下文对象的指针了。WDF如此这般主要目的是要封装WDFDEVICE对象，避免我们直接去访问其数据结构。

接着我们需要在调用WdfDeviceCreate创建设备对象时增加对对象属性(`WDF_OBJECT_ATTRIBUTES`)的设置。  
 - 先定义一个属性对象`WDF_OBJECT_ATTRIBUTES attributes;`  
 - 然后通过以下的宏初始化这个属性对象，同时通知FrameWork将我们定义的设备上下文对象(DEVICE_CONTEXT)插入到属性对象结构中。  
`WDF_OBJECT_ATTRIBUTES_INIT_CONTEXT_TYPE(&attributes, DEVICE_CONTEXT);`  
 - 最后在调用WdfDeviceCreate时我们就不会再象Step1中那样采用缺省的属性参数了，将我们先前定义并初始化好的属性对象attributes作为第二个参数传入。这样FrameWork就知道需要将我们的上下文对象和设备对象建立起联系。
`status = WdfDeviceCreate(&DeviceInit, &attributes, &device);`

## *为OSRFX2准备硬件IO*
FrameWork要求我们在另一个重要的回调函数[EvtDevicePrepareHardware]中对物理设备进行初始化操作以确保驱动程序可以访问硬件设备。比如：建立物理地址和虚拟地址之间的映射以便驱动可以访问系统分配给设备的地址等。EvtDevicePrepareHardware所进行的初始化操作应该是设备枚举后只执行一次的动作。对于其他需要在设备进入工作状态(D0)的初始化操作，我们建议放在[EvtDeviceD0Entry]中执行。

首先我们来看一下如何注册EvtDevicePrepareHardware。为了向FrameWork注册EvtDevicePrepareHardware，我们需要在[EvtDriverDeviceAdd]中调用[WdfDeviceCreate]之前对[WdfDeviceCreate]的DeviceInit参数进行初始化,主要是调用[WdfDeviceInitSetPnpPowerEventCallbacks]对DeviceInit设置与PnP以及电源管理相关的回调函数。示例代码如下：  
`WDF_PNPPOWER_EVENT_CALLBACKS pnpPowerCallbacks;`  
`WDF_PNPPOWER_EVENT_CALLBACKS_INIT(&pnpPowerCallbacks);`  
`pnpPowerCallbacks.EvtDevicePrepareHardware = EvtDevicePrepareHardware;`  
`WdfDeviceInitSetPnpPowerEventCallbacks(DeviceInit, &pnpPowerCallbacks);`  

为OSRFX2准备硬件IO的主要工作就是要在Step1创建FDO的基础上为OSRFX2的FDO关联一系列的[USB I/O Targets]。我们的驱动需要通过这些对象和真正的OSRFX2设备建立USB的会话并开展通讯。  

首先我们要创建一个USB设备对象。调用[WdfUsbTargetDeviceCreate]并将该函数创建的WDFUSBDEVICE对象保存在我们的上下文空间中以备后用。我们可以看到该API的第一个参数是WDFDEVICE。这样我们调用WdfUsbTargetDeviceCreate后WDF就将我们的设备对象FDO和USB对象Target联系起来了。  
`status = WdfUsbTargetDeviceCreate(Device,WDF_NO_OBJECT_ATTRIBUTES,&pDeviceContext->UsbDevice);`  

创建完USB设备对象后我们就需要对其进行配置。在这里我们必须要做的头一件事情就是要对设备选择一个配置(selects a USB configuration).一个配置定义了一个USB设备的一种工作模式，会暴露出几个接口(interfaces)，每个接口有几个端点(endpoints)，每个端点的类型，方向等等。总之只有选择了合适的配置我们才知道这个USB设备是如何工作的，从而我们才知道如何与该USB设备交流，不是吗？  
为了选择配置，我们先定义一个配置参数对象：  
`WDF_USB_DEVICE_SELECT_CONFIG_PARAMS configParams;`  
根据OSRFX2的定义，它是一个单接口的设备，所以我们将其初始化为一个单接口设备：  
`WDF_USB_DEVICE_SELECT_CONFIG_PARAMS_INIT_SINGLE_INTERFACE(&configParams);`  
最后我们将USB设备根据我们的需要选择好并创建USB接口对象并将该接口对象也存储在我们的上下文中备用。    
`status = WdfUsbTargetDeviceSelectConfig(pDeviceContext->UsbDevice,WDF_NO_OBJECT_ATTRIBUTES,&configParams);`  
`pDeviceContext->UsbInterface = configParams.Types.SingleInterface.ConfiguredUsbInterface; `  

## *为OSRFX2注册设备接口([device interface])*
KMDF驱动程序运行在内核态，为了让用户态的应用程序APP能够访问我们的驱动程序对象，主要是设备接口对象，驱动程序需要和操作系统协作在暴露出应用程序可以在用户态访问的接口，这就是[device interface]，俗称符号链接(symbolic link)。应用程序可以将设备接口的符号链接作为调用Win32 API的参数来和驱动打交道。更详细的描述可以参考[Using Device Interfaces]。

具体的操作可分如下几部：
首先要用[guidgen.exe]为OSRFX2的接口对象产生一个GUID:  
`DEFINE_GUID(GUID_DEVINTERFACE_OSRUSBFX2, // Generated using guidgen.exe`  
`   0x573e8c73, 0xcb4, 0x4471, 0xa1, 0xbf, 0xfa, 0xb2, 0x6c, 0x31, 0xd3, 0x84);`  
`// {573E8C73-0CB4-4471-A1BF-FAB26C31D384}`  

然后在EvtDeviceAdd回调函数中调用[WdfDeviceCreateDeviceInterface]即可  
`status = WdfDeviceCreateDeviceInterface(device,(LPGUID) &GUID_DEVINTERFACE_OSRUSBFX2,NULL);// Reference String`  

# Step3 - USB控制传输.

在继续Step3和Step4之前，我们要先理解一些KMDF驱动中处理I/O Requests相关的概念。详细的描述可以参考[Handling I/O Requests in KMDF Drivers]。

[Framework Request Objects] - I/O请求对象，是设计为表示I/O Manager发送给驱动的I/O请求。每个I/O请求封装了一个WDM的I/O Request packet,或者简称为IRP。基于WDF写的驱动一般不需要直接访问IRP结构，而是通过调用[Framework Request Objects]提供的成员函数对其进行操作。

[Framework Queue Objects] - I/O队列对象，是设计为表示用来缓存I/O请求对象的队列。为了简化驱动程序员编写WDM驱动的工作量和处理并行I/O事件，串行I/O事件的复杂性，WDF抽象出了I/O队列的概念来接收驱动需要处理的IO请求。对每个设备可以创建一个或多个队列。FrameWork为I/O队列类定义了一个回调函数的集合，WDF称之为[Request Handlers]。我们可以选取我们部分回调函数并加入我们自己的处理代码（类似C++中虚函数重载的概念）来处理我们感兴趣的事件，FrameWork也为队列类定义了一组操作成员函数供驱动操作队列。

## *注册I/O控制*

驱动程序的主要面向用户层的接口就是处理IO。用户程序通过调用DeviceIoControl，传入I/O控制码（I/O Control Codes）来通知驱动响应特定的请求。详细的描述可以参考[Using I/O Control Codes]。 定义I/O控制码很简单，有一个Windows定义的宏`CTL_CODE`.  
`#define IOCTL_INDEX                     0x800`  
`#define FILE_DEVICE_OSRUSBFX2          0x65500`  
`#define USBFX2LK_SET_BARGRAPH_DISPLAY 0xD8`  
`#define IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY CTL_CODE(FILE_DEVICE_OSRUSBFX2, IOCTL_INDEX + 5, METHOD_BUFFERED, FILE_WRITE_ACCESS)`  
和大多数驱动一样，OSRFX2的驱动在EvtDriverDeviceAdd回调函数中创建自己的I/O队列.  
首先定义一个类型为`WDF_IO_QUEUE_CONFIG`的对象来设置队列的属性特征。  
`WDF_IO_QUEUE_CONFIG                 ioQueueConfig;`  
我们可以根据自己的需要对不同的IO请求创建不同功能的队列来进行处理。在Step3这个例子里为简单起见我们只创建了一个缺省的队列。所谓缺省队列就是说，一旦你为你的设备创建了一个缺省的I/O队列，FrameWork会把所有的I/O请求都放到这个缺省队列里来进行处理，除非你为这个I/O请求创建了其他的处理队列。

初始化缺省队列我们可以调用WDF的宏`WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE `。第二个参数传入WdfIoQueueDispatchParallel表明这个队列不会因为我们的I/O处理函数而阻塞。有关I/O分发的机制可以参考[Dispatching Methods for I/O Requests]。  
`WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE(&ioQueueConfig, WdfIoQueueDispatchParallel);`  
注册IO Control的回调函数入口[EvtIoDeviceControl]：  
`ioQueueConfig.EvtIoDeviceControl = EvtIoDeviceControl;`  
OK, 接下来我们就可以调用[WdfIoQueueCreate]来创建这个队列了。
`status = WdfIoQueueCreate(device,&ioQueueConfig,WDF_NO_OBJECT_ATTRIBUTES,WDF_NO_HANDLE);`  

以上工作做好后，当应用程序调用DeviceIoControl并传入`IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY`来设置LED bar的亮和灭的时候，我们的驱动程序的EvtIoDeviceControl函数就会被FrameWork激活回调。具体的操作就看我们在EvtIoDeviceControl里的处理代码了。因为应用APP在调用DeviceIoControl的时候除了指明IO控制码还需要给定一些其他的参数。对于设置LED bar来说就是要指定点亮或者熄灭8盏灯中的哪一盏。下面我们就来看看用户态的APP和内核态的驱动是怎样传递这些内容的以及我们的驱动又是怎样将这些内容传递给真正的执行体--"设备"的。

## *处理I/O控制*
现在我们来看一下EvtIoDeviceControl中是如何处理应用发起的`IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY`请求的。
[EvtIoDeviceControl]一共有五个参数，都是入(`_In_`)参。因为`IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY`操作目的是设置设备的LED bar，所以相对于驱动，没有需要传出给应用的内容，所以没有用到OutputBufferLength；应用有要传给驱动设置LEDbar的内容，大小是一个UCHAR，也就是一个字节，所以InputBufferLength是有效的。我们可以在EvtIoDeviceControl里判断一下，如果太小说明有问题。  
`if(InputBufferLength < sizeof(UCHAR)) {`  
...  
`}`  
接下来我们就可以在驱动里尝试得到应用程序传入的参数内容了。  
`WDFMEMORY                           memory;`  
`status = WdfRequestRetrieveInputMemory(Request, &memory);`  
OS已经将用户态的数据封装为IRP，FrameWork进而将IRP封装为I/O请求，我们现在要做的就是从I/O请求中得到储存数据的内存句柄，[WdfRequestRetrieveInputMemory]干的就是这个。  
得到了数据，这里我们不需要额外的处理，所以接下来就是准备发送给设备了。
PC和OSRFX2设备之间设置LEDbar的命令采用的是EP0上的Setup传输。  
`WDF_USB_CONTROL_SETUP_PACKET        controlSetupPacket;`  
`WDF_USB_CONTROL_SETUP_PACKET_INIT_VENDOR(&controlSetupPacket,`  
`                                         BmRequestHostToDevice,`  
`                                         BmRequestToDevice,`  
`                                         USBFX2LK_SET_BARGRAPH_DISPLAY,`  
`                                         0,`  
`                                         0);`  
最后的操作就是初始化其他的参数后调用最终的[WdfUsbTargetDeviceSendControlTransferSynchronously]将数据发送的USB总线上，这样设备就可以接收到了。这里我们采用的是同步发送并设置了超时。


# Step4 - USB批量传输.




参考：  
http://www.ituring.com.cn/article/554  
http://channel9.msdn.com/Shows/Going+Deep/Doron-Holan-Kernel-Mode-Driver-Framework

[Framework Object Context Space]: http://msdn.microsoft.com/en-us/library/ff542873(v=vs.85).aspx
[device interface]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff556277(v=vs.85).aspx#wdkgloss.device_interface
[USB I/O Targets]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff544752(v=vs.85).aspx
[Using Device Interfaces]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff545432(v=vs.85).aspx
[guidgen.exe]: http://msdn.microsoft.com/en-us/library/ms241442(v=vs.80).aspx
[Framework Queue Objects]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff542922(v=vs.85).aspx
[Using I/O Control Codes]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff565406(v=vs.85).aspx
[Dispatching Methods for I/O Requests]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff540800(v=vs.85).aspx
[Handling I/O Requests in KMDF Drivers]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff543296(v=vs.85).aspx
[Framework Request Objects]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff542962(v=vs.85).aspx
[Request Handlers]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff544583(v=vs.85).aspx


[WdfDriverCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff547175(v=vs.85).aspx
[WdfDeviceCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff545926(v=vs.85).aspx
[WdfDeviceInitSetPnpPowerEventCallbacks]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff546135(v=vs.85).aspx
[WdfUsbTargetDeviceCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff550077(v=vs.85).aspx
[WdfDeviceCreateDeviceInterface]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff545935(v=vs.85).aspx
[WdfIoQueueCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff547401(v=vs.85).aspx
[WdfRequestRetrieveInputMemory]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff550015(v=vs.85).aspx
[WdfUsbTargetDeviceSendControlTransferSynchronously]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff550104(v=vs.85).aspx
[WdfRequestCompleteWithInformation]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff549948(v=vs.85).aspx

[DriverEntry]: http://msdn.microsoft.com/zh-cn/library/windows/hardware/ff544113(v=vs.85).aspx
[EvtDriverDeviceAdd]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff541693(v=vs.85).aspx
[EvtDevicePrepareHardware]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff540880(v=vs.85).aspx
[EvtDeviceD0Entry]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff540848(v=vs.85).aspx
[EvtIoDeviceControl]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff541758(v=vs.85).aspx