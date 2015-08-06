第一节 查找设备
===============

1.1 查找设备信息
----------------

调用函数 FindFirstDevice() 可以搜索设备信息，同时填充 DEVMGR_DEVICE_INFORMATION 结构体。

1. 函数原型如下：

.. code-block:: C++

    HANDLE FindFirstDevice(
      DeviceSearchType searchType,
      LPCVOID pvSearchParam,
      PDEVMGR_DEVICE_INFORMATION pdi
    );

2. 参数说明
   
   +---------------+----------------------------------------------------+
   | 参数名        | 说明                                               |
   +===============+====================================================+
   | searchType    | 设备搜索方法                                       |
   +---------------+----------------------------------------------------+
   | pvSearchParam | 字符串指针，GUID 指针或者一个激活的句柄            |
   +---------------+----------------------------------------------------+
   | pdi           | DEVMGR_DEVICE_INFORMATION 结构体，用于获取设备信息 |
   +---------------+----------------------------------------------------+
   | 返回值        | 成功则返回句柄；失败返回 INVALID_HANDLE_VALUE      |
   +---------------+----------------------------------------------------+

3. 备注
   
   该函数的第二个参数和第一个参数是有关联的。
   具体关联见下表：

   +--------------------------+------------------------------------------------+
   | searchType               | pvSearchParam                                  |
   +==========================+================================================+
   | DeviceSearchByLegacyName | L"COM*" for all COMx: devices                  |
   +--------------------------+------------------------------------------------+
   | DeviceSearchByDeviceName | L"COM*" for all COMx devices                   |
   +--------------------------+------------------------------------------------+
   | DeviceSearchByBusName    | L"PCI_0_3*" for PCI_0_3_0, PCI_0_3_1 and so on |
   +--------------------------+------------------------------------------------+
   | DeviceSearchByGuid       | Pointer to a GUID                              |
   +--------------------------+------------------------------------------------+
   | DeviceSearchByParent     | Activation handle value from ActivateDeviceEx. |
   +--------------------------+------------------------------------------------+

4. 实例：
   
   .. code-block:: C++
 
       DEVMGR_DEVICE_INFORMATION deviceInfo;

       ZeroMemory(&deviceInfo, sizeof(DEVMGR_DEVICE_INFORMATION));
       deviceInfo.dwSize = sizeof(DEVMGR_DEVICE_INFORMATION);
   
       HANDLE drive = FindFirstDevice(DeviceSearchByDeviceName, L"WED1", &deviceInfo);

1.2 查找周边蓝牙设备
--------------------

需要用到的接口:

* WSAStartup()
* WSALookupServiceBegin()
* WSALookupServiceNext()
* WSALookupServiceEnd()
* WSACleanup()

因为查询周边蓝牙设备需要使用到 `WinSock2.h` 和对应的动态库，
所以必须调用 WSAStartup() 和 WSACleanup() 函数进行动态库内的初始化操作。

.. note:: WSAStartup() 的第一个参数是 0x0100，即 MAKEWORD(1,0)，不是 0x0202

``WSALookupServiceBegin()``

参数列表:

+---------------------+----------+---------------+------------------------------+
| 参数名              | 输入输出 | 类型          | 说明                         |
+=====================+==========+===============+==============================+
| lpqsRestrictions    | IN       | LPWSAQUERYSET | 搜索参数                     |
+---------------------+----------+---------------+------------------------------+
| dwControlFlags [1]_ | IN       | DWORD         | 取 LUPCONTAINER 即可         |
+---------------------+----------+---------------+------------------------------+
| lphLookup           | OUT      | LPHANDLE      | 输出句柄                     |
+---------------------+----------+---------------+------------------------------+
| return              | OUT      | DWORD         | 系统错误码或者 ERROR_SUCCESS |
+---------------------+----------+---------------+------------------------------+

WSAQUERYSET 是一个搜索参数的结构体，你可以通过它来限定搜索规则方式。
需要设定的两个成员是 dwSize 和 dwNameSpace，它们的值可以如下设置：

.. code-block:: C++

    WSAQUERYSET querySet = { 0 }; // 不用的字段都归零

    querySet.dwSize = sizeof(WSAQUERYSET);
    querySet.dwNameSpace = NS_BTH

.. note:: WSALookupServiceBegin() 只是开始搜索，获取设备结果要通过 WSALookupServiceNext() 实现

输出的字段包括：

* lpszServiceInstanceName     用于存储蓝牙设备名
* lpVersion                   版本号
* dwNumberOfCsAddrs           地址个数
* lpcsaBuffer                 存储地址信息

``WSALookupServiceNext()``

参数列表:

+------------------+----------+---------------+--------------------------------------------------------+
| 参数名           | 输入输出 | 类型          | 说明                                                   |
+==================+==========+===============+========================================================+
| hLookup          | IN       | HANDLE        | 由 WSALookupServiceBegin() 生成                        |
+------------------+----------+---------------+--------------------------------------------------------+
| dwControlFlags   | IN       | DWORD         | 具体由需求决定，如果要获取地址，则可取 LUP_RETURN_ADDR |
+------------------+----------+---------------+--------------------------------------------------------+
| lpdwBufferLength | IN       | DWORD         | 存储结果的 buffer 长度                                 |
+------------------+----------+---------------+--------------------------------------------------------+
| lpqsResults      | OUT      | LPWSAQUERYSET | 存储结果集                                             |
+------------------+----------+---------------+--------------------------------------------------------+
| return           | OUT      | DWORD         | 系统错误码或者 ERROR_SUCCESS                           |
+------------------+----------+---------------+--------------------------------------------------------+

需要注意的是 lpqsResults 的类别是 LPWSAQUERYSET，但其内存空间最好保持足够大，这样不会有问题。
代码示例：

.. code-block:: C++

    char buf[5000];
    LPWSAQUERYSET querySet = (LPWSAQUERYSET)buf;
    ZeroMemory(querySet, sizeof(WSAQUERYSET));
    querySet.dwSize = sizeof(WSAQUERYSET);
    querySet.dwNameSpace = NS_BTH;

    int err;
    err = WSALookupServiceNext(handle, LUP_RETURN_ADDR, sizeof(buf), querySet);
    if (err != ERROR_SUCCESS)
    {
        cout << "error code: " << err << endl;
    }

    BT_ADDR btAddr = ((SOCKADDR_BTH*)querySet-UINT8>lpcsaBuffer->UINT8.lpSockaddr)->btAddr;
    UINT8* pt = (UINT8*)&btAddr;
    
    char mac[18];
    sprintf(mac, "%02x:%02x:%02x:%02x:%02x:%02x",
        pt[5], pt[4], pt[3], pt[2], pt[1], pt[0]);

``WSALookupServiceEnd()``

结束查询。该函数只有一个参数，就是 WSALookupServiceBegin() 得到的 HANDLE

.. [1] 控制码分为两种，一种是行为码，只用在 WSALookupServiceNext() 函数里，当前调用有效；
 另一种是类型码，可以放在 WSALookupServiceBegin() 里，一直有效。
 例如：LUPCONTAINERS 就是类型码，而 LUP_RETURN_ADDR 是行为码。