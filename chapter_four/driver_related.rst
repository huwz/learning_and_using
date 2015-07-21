第四章 驱动相关
===============

4.1 查找设备信息
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

