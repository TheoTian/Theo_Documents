## Android反编译小工具
#### AndroidManifest.xml
当解压apk之后AndroidManifest.xml已经转为二进制码，文本打开为乱码，使用AXMLPrinter2.jar还原。

```
java -jar AXMLPrinter2.jar AndroidManifest.xml > AndroidManifest.txt
```

