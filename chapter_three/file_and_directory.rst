第一节 文件和文件夹
===================

1.1 判断文件/文件夹是否存在
---------------------------

方法比较多，在 windows 或者 wince 平台下都能使用的方法，列举如下：

1. 使用 access() 函数
2. 使用 GetFileAttributes()
   
   判断文件是否存在：

   .. code-block:: C++
       :linenos:

       wchar_t* path = L"D:\\1.txt";

       if (GetFileAttributes(path) == 0xFFFFFFFF)
       {
           std::wcout << L"File " << path << L"not exist" << endl;
       }

   .. note:: 这种方法也可以用来判断驱动（设备描述符）是否加载
    
   判断文件夹是否存在：

   .. code-block:: C++
       :linenos:

       wchar_t* path = L"D:\\tmp\";

       if (GetFileAttributes(path) == 0xFFFFFFFF)
       {
           std::wcout << L"Directory " << path << L"not exist" << endl;
       }

3. 使用 GetFirstFile()

   判断文件是否存在：

   .. code-block:: C++
       :linenos:

       wchar_t file = L"D:\\1.txt";
       if (GetFirstFile(file, NULL) == INVALID_HANDLE_VALUE)
       {
           std::wcout << L"File " << path << L"not exist" << endl;
       }

    判断文件夹是否存在：
    
    .. code-block:: C++
        :linenos:

        wchar_t file = L"D:\\tmp\\*";
        if (GetFirstFile(file, NULL) == INVALID_HANDLE_VALUE)
        {
            std::wcout << L"Directory " << path << L"not exist" << endl;
        }

1.3 删除文件(夹)
----------------

删除文件：

.. code-block:: C++

    if (!DeleteFile(filename))
    {
        cout << "Delete file FAILED, code :" << GetLastError() << endl;
    }