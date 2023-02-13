网络OSI（Open System Interconnection）模型，由国际标准化组织(ISO)提出：

![OSI网络模型-1](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E4%BA%92%E8%81%94%E7%BD%91%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84/resources/OSI-1.png)\
![OSI网络模型-2](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E4%BA%92%E8%81%94%E7%BD%91%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84/resources/OSI-2.png)

在数据发送到另一层时，都要分成数据包，数据包的创建过程是从OSI模型的应用层开始的。跨网络传输的信息要从应用层开始，往下依次穿过各层。每层都对数据包进行重新组装，以增加自己的信息。\
![OSI-package](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E4%BA%92%E8%81%94%E7%BD%91%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84/resources/OSI-package.png)\
![OSI-unpackage](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E4%BA%92%E8%81%94%E7%BD%91%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84/resources/OSI-unpackage.png)




