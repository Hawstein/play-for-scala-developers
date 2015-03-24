# Messages externalisation and i18n

##ָ��Ӧ��֧������
����԰�**ISO 639-2** ��׼���Դ������**ISO 3166-1 alpha-2**��׼���Ҵ�����ָ��Ӧ��֧�����ԣ�Ʃ��`fr`����`en-US`��

����ָ����Ϣ�����Ӧ�õ� `conf/application.conf`�ļ��У�

```scala
application.langs="en,en-US,fr"
```

##�ⲿ����Ϣ��Externalizing messages��

�������`conf/messages.xxx`�ļ����ⲿ����externalize����Ϣ��

`conf/messages`�ļ�Ĭ�������������ԡ�������������ָ����������Ӧ���ļ���Ʃ��`conf/messages.fr`��`conf/messages.en-US`��

���������`play.api.i18n.Messages`������(retrieve)��Ϣ��

```scala
val title = Messages("home.title")
```

���й��ʻ���ص�API����ʽ����`play.api.i18n.Lang`���������в�����ֵ���ݵ�ǰ����������������Լ���ʽ��ָ����

```scala
val title = Messages("home.title")(Lang("fr"))
```

**ע��:**������ڵ�ǰ��Χ����һ����ʽ���󣬳�������ͷ���� `Accept-Language`��Ӧ��֧������ƥ��һ����Ȼ����ʽ�ط��ظ���һ��`Lang`��������Ӧ����������ģ���ļ������`Lang`������ `@()(implicit lang: Lang)`��


##��Ϣ��ʽ

��Ϣʹ�� `java.text.MessageFormat`���������ʽ��Ʃ�������Ϣ�������壺

```
files.summary=The disk {1} contains {0} file(s).
```
��ô���������ָ��������

```
Messages("files.summary", d.files.length, d.name)
```

##���ڵ�����


��������ʹ��`java.text.MessageFormat`��������Ϣ��ʽ������Ҫע����ǵ����Żᱻ��Ϊת���ַ���

Ʃ���㶨��������һ����Ϣ��

���磬����ܻᶨ������һ����Ϣ��

```scala
info.error=You aren''t logged in!
```

```scala
example.formatting=When using MessageFormat, '''{0}''' is replaced with the first parameter.
```

��Ӧ�������½����

```scala
Messages("info.error") == "You aren't logged in!"
```

```scala
Messages("example.formatting") == "When using MessageFormat, '{0}' is replaced with the first parameter."
```

##��HTTP��������ȡ֧������

����ԣ����������Ӹ�����HTTP������ȡ��֧�����ԣ�

```
def index = Action { request =>
  Ok("Languages: " + request.acceptLanguages.map(_.code).mkString(", "))
}
```

**��һ�ڣ�** [The application Global object](../ch13/ScalaGlobal.md)