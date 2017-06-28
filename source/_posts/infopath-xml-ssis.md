---
title: SSIS提取InfoPath XML数据示例
tags:
  - ETL
  - InfoPath
  - SSIS
  - XML
id: 1359
categories:
  - 商业智能
date: 2016-11-01 09:06:54
---

引言
最近碰到一个看似简单的项目，要提取InfoPath 的表单的数据。关于InfoPath 表单还是有点认识的，表单是以XML格式存储的，模板文件后缀名xsn，应该很容易从模板文件提取schema结构读取XML数据。虽然InfoPath作为一个产品将在2023年永远的退出历史，但是作为表单工具对于办公室信息工作人员来说还是比较受欢迎的。因此一些企业积累了不少的表单数据需要转成其他应用可读的数据。这个项目就是要把表单数据通过SSIS这个ETL工具导入数据库以供后续分析。
**关于InfoPath**,[互动百科](http://www.baike.com/wiki/infopath)是这样描述的。
> InfoPath是微软Office 2003家族中引入的成员，最终的正式版本为InfoPath2013，该版本支持在线填写表单。> 
> InfoPath是企业级搜集信息和制作表单的工具，将很多的界面控件集成在该工具中，为企业开发表单搜集系统提供了极大的方便。> 
> InfoPath文件的后缀名是.xml，可见InfoPath是基于XML技术的。> 
> 作为一个数据存储中间层技术，InfoPath提供大量常用控件，如：Date Picker、文本框、可选节、重复节等，同时提供很多表格页面设计工具。> 
> 开发人员可以为每个控件设置相应的数据有效性规则或数学公式。> 
> 如果InfoPath仅能做到上述功能，那么我们是可以用Excel做的表单代替InfoPath的，最重要的功能，就是InfoPath提供和数据库和Web服务之间的连接。> 
> 2014年1月31日，微软office官方博客宣布，InfoPath2013为最后的桌面客户端版本，InfoPath桌面软件和服务器产品的Lifecycle支持都会到2023年4月。

**InfoPath的文件组成**
InfoPath通常有一个以xsn为后缀名的模板文件，一个正常的InfoPath模板文件包含以下文件
manifest.xsf——InfoPath的定义文件，定义整个InfoPath的组成结构，里面会指定该InfoPath的其它文件，并且必须是InfoPath中的第一个文件。
myschema.xsd——InfoPath的架构文件，定义该InfoPath里面的包含的域和域的类型
sampledata.xml——包含InfoPath的域
template.xml——新建表单的时候进行实例化InfoPath
view1.xsl——存储InfoPath中视图的文件
一个典型的InfoPath XML如下所示，

    &lt;?xml version="1.0"?&gt;&lt;?mso-infoPathSolution productVersion="14.0.0" PIVersion="1.0.0.0" href="http://intranet/PO/template.xsn" name="urn:schemas-microsoft-com:office:infopath:PO:-myXSD-2009-03-06T03-16-15" solutionVersion="1.0.0.1083" ?&gt;&lt;?mso-application progid="InfoPath.Document" versionProgid="InfoPath.Document.2"?&gt;&lt;?mso-infoPath-file-attachment-present?&gt;&lt;my:myFields xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dfs="http://schemas.microsoft.com/office/infopath/2003/dataFormSolution" xmlns:s0="http://microsoft.com/webservices/SharePointPortalServer/UserProfileService" xmlns:s1="http://microsoft.com/wsdl/types/" xmlns:mime="http://schemas.xmlsoap.org/wsdl/mime/" xmlns:tm="http://microsoft.com/wsdl/mime/textMatching/" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:http="http://schemas.xmlsoap.org/wsdl/http/" xmlns:ns1="http://schemas.xmlsoap.org/wsdl/" xmlns:my="http://schemas.microsoft.com/office/infopath/2003/myXSD/2009-03-06T03:16:15" xmlns:xd="http://schemas.microsoft.com/office/infopath/2003" xml:lang="en-us"&gt;
    	&lt;my:PONo&gt;201609223&lt;/my:PONo&gt;
    	&lt;my:POVendor&gt;1&lt;/my:POVendor&gt;
    	&lt;my:PRItemD&gt;
    			&lt;my:ItemNo&gt;11223344&lt;/my:ItemNo&gt;
    			&lt;my:ItemDesc&gt;11223344 desc&lt;/my:ItemDesc&gt;
    			&lt;my:ItemQty&gt;1&lt;/my:ItemQty&gt;
    			&lt;my:ItemPrice&gt;1203.2&lt;/my:ItemPrice&gt;
    			&lt;my:ItemAmt&gt;1203.2&lt;/my:ItemAmt&gt;
    	&lt;/my:PRItemD&gt;
    &lt;/my:myFields&gt;`</pre>
    从以上我们看到InfoPath除了前面的定义外，InfoPath的数据确实是XML格式的。

    **问题的产生**
    但是当我们使用SSIS读取这个InfoPath表单时却出现了The XML contains multiple namespaces的错误。
    [![2016-10-25_16-47-37](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-25_16-47-37.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-25_16-47-37.jpg)
    这个问题怎么产生的呢？如何解决这个错误呢？
    分析InfoPath的XML文件，我们看到很多xmlns开头的namespaces的定义，对于存储的数据来说，我们其实只要保留xmlns:xsi这个namespace就可以了。
    因此我们需要用XSLT解析InfoPath的XML文件并去掉文件中的InfoPath额外信息
    <pre>`&lt;?xml version="1.0" encoding="utf-8" ?&gt; 
    &lt;xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform"&gt;
      &lt;xsl:output method="xml" indent="yes" /&gt; 
      &lt;xsl:template match="/|comment()|processing-instruction()"&gt;
        &lt;xsl:copy&gt;
          &lt;xsl:apply-templates /&gt; 
        &lt;/xsl:copy&gt;
      &lt;/xsl:template&gt;
      &lt;xsl:template match="*"&gt;             
        &lt;xsl:element name="{local-name()}"&gt;
          &lt;xsl:apply-templates select="@*|node()" /&gt; 
        &lt;/xsl:element&gt;
      &lt;/xsl:template&gt;
      &lt;xsl:template match="@*"&gt;
        &lt;xsl:attribute name="{local-name()}"&gt;
          &lt;xsl:value-of select="." /&gt; 
        &lt;/xsl:attribute&gt;
      &lt;/xsl:template&gt;
    &lt;/xsl:stylesheet&gt;`</pre>

    生成新的xml数据如下
    <pre>`&lt;?xml version="1.0" encoding="utf-8"?&gt;&lt;?mso-infoPathSolution productVersion="14.0.0" PIVersion="1.0.0.0" href="http://intranet/PO/template.xsn" name="urn:schemas-microsoft-com:office:infopath:PO:-myXSD-2009-03-06T03-16-15" solutionVersion="1.0.0.1083" ?&gt;&lt;?mso-application progid="InfoPath.Document" versionProgid="InfoPath.Document.2"?&gt;&lt;?mso-infoPath-file-attachment-present?&gt;
    &lt;myFields lang="en-us"&gt;
    	&lt;PONo&gt;201609223&lt;/PONo&gt;
    	&lt;POVendor&gt;1&lt;/POVendor&gt;
    	&lt;PRItemD&gt;
    		&lt;ItemNo&gt;11223344&lt;/ItemNo&gt;
    		&lt;ItemDesc&gt;11223344 desc&lt;/ItemDesc&gt;
    		&lt;ItemQty&gt;1&lt;/ItemQty&gt;
    		&lt;ItemPrice&gt;1203.2&lt;/ItemPrice&gt;
    		&lt;ItemAmt&gt;1203.2&lt;/ItemAmt&gt;
    	&lt;/PRItemD&gt;
    &lt;/myFields&gt;`</pre>

    但是这样还是不能提取数据,我们看到SSIS 没有读取PRNO，PRVENDOR栏位信息，这是因为这些信息没有包含在一个节点里。
    [![2016-10-26_16-41-55](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-26_16-41-55.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-26_16-41-55.jpg)
    这就需要我们再次使用XSLT格式化XML文件。
    <pre>`&lt;?xml version="1.0" encoding="utf-8" ?&gt; 
    &lt;xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform"&gt;
      &lt;xsl:output method="xml" indent="yes" /&gt; 
      &lt;xsl:template match="/myFields"&gt;  
      &lt;PR&gt;
    	    &lt;PRHead&gt;
    			&lt;PONo&gt;&lt;xsl:value-of select="PONo"/&gt;&lt;/PONo&gt;
    			&lt;POVendor&gt;&lt;xsl:value-of select="POVendor"/&gt;&lt;/POVendor&gt;	
    		&lt;/PRHead&gt;
    	&lt;xsl:for-each select="PRItemD"&gt;
            &lt;PRItemD&gt;
                &lt;ItemNo&gt;&lt;xsl:value-of select="ItemNo"/&gt;&lt;/ItemNo&gt;
                &lt;ItemDesc&gt;&lt;xsl:value-of select="ItemDesc"/&gt;&lt;/ItemDesc&gt;
                &lt;ItemQty&gt;&lt;xsl:value-of select="ItemQty"/&gt;&lt;/ItemQty&gt;
                &lt;ItemPrice&gt;&lt;xsl:value-of select="ItemPrice"/&gt;&lt;/ItemPrice&gt;
                &lt;ItemAmt&gt;&lt;xsl:value-of select="ItemAmt"/&gt;&lt;/ItemAmt&gt;
            &lt;/PRItemD&gt;
        &lt;/xsl:for-each&gt;
        &lt;/PR&gt;
      &lt;/xsl:template&gt;
    &lt;/xsl:stylesheet&gt;`</pre>
    通过第二次转换，我们获得了完全符合XML标准的文件
    <pre>`&lt;?xml version="1.0" encoding="utf-8"?&gt;
    &lt;PR&gt;
      &lt;PRHead&gt;
        &lt;PONo&gt;201609223&lt;/PONo&gt;
        &lt;POVendor&gt;1&lt;/POVendor&gt;
      &lt;/PRHead&gt;
      &lt;PRItemD&gt;
        &lt;ItemNo&gt;11223344&lt;/ItemNo&gt;
        &lt;ItemDesc&gt;11223344 desc&lt;/ItemDesc&gt;
        &lt;ItemQty&gt;1&lt;/ItemQty&gt;
        &lt;ItemPrice&gt;1203.2&lt;/ItemPrice&gt;
        &lt;ItemAmt&gt;1203.2&lt;/ItemAmt&gt;
      &lt;/PRItemD&gt;
    &lt;/PR&gt;`</pre>

    接下来在Data Flow Task里添加XML source组件。
    点击生成XSD按钮，可以自动从XML数据中获得，你可以需要对字段类型进行修改，因为默认导出的不一致符合字段的要求。
    <pre>`&lt;?xml version="1.0"?&gt;
    &lt;xs:schema attributeFormDefault="unqualified" elementFormDefault="qualified" xmlns:xs="http://www.w3.org/2001/XMLSchema"&gt;
      &lt;xs:element name="PR"&gt;
        &lt;xs:complexType&gt;
          &lt;xs:sequence&gt;
            &lt;xs:element minOccurs="0" name="PRHead"&gt;
              &lt;xs:complexType&gt;
                &lt;xs:sequence&gt;
                  &lt;xs:element minOccurs="0" name="PrNo" /&gt;
                  &lt;xs:element minOccurs="0" name="POVendor" type="xs:unsignedByte" /&gt;
                &lt;/xs:sequence&gt;
              &lt;/xs:complexType&gt;
            &lt;/xs:element&gt;
            &lt;xs:element minOccurs="0" name="PRItemD"&gt;
              &lt;xs:complexType&gt;
                &lt;xs:sequence&gt;
                  &lt;xs:element minOccurs="0" name="ItemNo" type="xs:unsignedInt" /&gt;
                  &lt;xs:element minOccurs="0" name="ItemDesc" type="xs:string" /&gt;
                  &lt;xs:element minOccurs="0" name="ItemQty" type="xs:unsignedByte" /&gt;
                  &lt;xs:element minOccurs="0" name="ItemPrice" type="xs:decimal" /&gt;
                  &lt;xs:element minOccurs="0" name="ItemAmt" type="xs:decimal" /&gt;
                &lt;/xs:sequence&gt;
              &lt;/xs:complexType&gt;
            &lt;/xs:element&gt;
          &lt;/xs:sequence&gt;
        &lt;/xs:complexType&gt;
      &lt;/xs:element&gt;
    &lt;/xs:schema&gt;

通过xsd我们读取了我们需要的数据。如图，我们同时看到了head和detail分组。
[![2016-11-01_9-05-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-01_9-05-36.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/2016-11-01_9-05-36.jpg)
接下来我们就可以指定输出文件了，为简单起见，示列文件我们输出到文本文件，根据项目需要，可以输出至数据库。如图，Data flow任务，这里我们分别输出head和detail到不同的文件。
[![2016-10-30_15-23-48](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_15-23-48.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_15-23-48.jpg)
整个package运行如下。
[![2016-10-30_15-26-22](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_15-26-22.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/2016-10-30_15-26-22.jpg)
对于需要处理大量Infopath XML 数据，只要把以上组件放在一个Foreach Loop Container里就可以了。附示列package包[下载](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/10/InfoPathXML_Sample.7z)