---
layout: post
title: XML Serialization in .NET Core
---

Sometimes, you have to parse XML. Yes, JSON exists, and you should use it, but tell that to your good friend the Legacy Platform. Anyway, there's two things you can use for this. 

1. The sensibly-named [XmlSerializer](https://docs.microsoft.com/en-us/dotnet/api/system.xml.serialization.xmlserializer?view=netframework-4.7.2).
2. The less obvious [DataContractSerializer](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.serialization.datacontractserializer?view=netframework-4.7.2).

They both have problems, depending on what you're trying to do, let's chat first about XmlSerializer.

## XmlSerializer
The main problem you're going to hit with this - by which I mean the problem that I hit - is that for something to be serialized, it has to have a public setter. Which means no constructor-only setting, no immutable pattern. As far as I understand, this is because the library expects to be able to deserialize whatever it's serializing, and needs to reconstruct the object that way. This is annoying if you just need to serialize a class and hurl it into the void for something else to worry about. 
There's a danger here that you might have an empty setter, which will leave you with the worst of all possible worlds: it won't set the property, it won't tell you that nothing is happening, and it won't even serialize it. Enjoy your frustration. 

`XmlSerializer` also has a quirk that means it includes line breaks and whitespace when serializing, so you will have to use an `XmlWriterSettings` object if you want to avoid that. 

This library also requires a parameterless constructor, but it can be `private`, so that's fine.

## DataContractSerializer
Your - that is, again, my - problem with this is the order the properties come in has to be either alphabetical, or specified. The default for deserialization is that it expects the incoming XML object to have all its properties in alphabetical order, or you have to put an attribute on it stating the order. This isn't very flexible, particularly if you don't control the object coming in because it's from a Legacy Platform written in COBOL and Sellotape. 
This library is cool with having private setters though, so if the order doesn't bother you, go wild. 

The answer, or at least what I ended up doing, is a bit hacky, but such is life with XML. Use a public setter, but throw an exception if anyone tries to use it. Let's get some code. 

## Code samples
These are ready to run in [Linqpad](https://www.linqpad.net/), if you want to use something else, you'll have to dump `Dump()`. This is your basic serialization/deserialization if you just want something to work.

### Using XmlSerializer

```
async Task Main()
{
	var memoryStream = new MemoryStream();
	var serializer = new XmlSerializer(typeof(SerializeMe));
	var content = new SerializeMe("zero", "one");
	
	using (var sw = new StreamWriter(memoryStream, new UTF8Encoding(false), 1024, true))
	{
		serializer.Serialize(XmlWriter.Create(sw), content);
		await memoryStream.FlushAsync();
	}

	memoryStream.Seek(0, SeekOrigin.Begin);
	var httpContent = new StreamContent(memoryStream);
	
	await httpContent.ReadAsStringAsync().Dump();
}

public class SerializeMe
{
	private SerializeMe() {}
	public SerializeMe(string zero, string one)
	{
		PropertyZero = zero;
		PropertyOne = one;
	}

	[XmlElement("one")]
	public string PropertyOne { get;  set; }

	[XmlElement("zero")]
	public string PropertyZero { get;  set; }
}
```

### Using DataContractSerializer
```
async Task Main()
{
	var content = new SerializeMe("zero", "one");
	
	var serializer = new DataContractSerializer(typeof(SerializeMe));
	var memoryStream = new MemoryStream();

	serializer.WriteObject(memoryStream, content);

	memoryStream.Position = 0;
	
	var httpContent = new StreamContent(memoryStream);
	var streamContent = await httpContent.ReadAsStreamAsync();
	
	var response = serializer.ReadObject(XmlReader.Create(streamContent));
	response.Dump();
}

[DataContract(Name = "request")]
public class SerializeMe
{

	public SerializeMe(string zero, string one)
	{
		PropertyZero = zero;
		PropertyOne = one;
	}

	[DataMember(Name = "one")]
	public string PropertyOne { get; private set; }

	[DataMember(Name = "zero")]
	public string PropertyZero { get; private set; }
}
```

## The final outcome
I went with `XmlSerializer` in the end, with the setter hack mentioned above. Here's the final thing:

```
using System;
using System.IO;
using System.Text;
using System.Xml;
using System.Xml.Serialization;

namespace Example.Colm.SerializeStuff
{
    class XmlStreamSerializer
    {
        public Stream Serialize<T>(T content)
        {
            if (content == null)
                throw new ArgumentNullException(nameof(content));

            var settings = new XmlWriterSettings
            {
                Indent = false,
                NewLineHandling = NewLineHandling.None
            };

            var serializer = new XmlSerializer(typeof(T));

            var ms = new MemoryStream();
            using (var sw = new StreamWriter(ms, Encoding.UTF8, 1024, true))
            using (var xmlWriter = XmlWriter.Create(sw, settings))
            {
                serializer.Serialize(xmlWriter, content);
                xmlWriter.Flush();
            }

            ms.Seek(0, SeekOrigin.Begin);
            return ms;
        }

        public T Deserialize<T>(Stream stream)
        {
            if (stream == null)
                throw new ArgumentNullException(nameof(stream));

            using (stream)
            {
                var serializer = new XmlSerializer(typeof(T));
                return (T)serializer.Deserialize(XmlReader.Create(stream));
            }
        }
    }
}
```