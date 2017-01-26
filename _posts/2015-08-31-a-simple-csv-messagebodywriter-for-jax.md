---
permalink: /:categories/:year/:month/:title.html
layout: post
title: A simple CSV MessageBodyWriter for JAX-RS using Jackson
date: '2015-08-31T14:19:00.000-07:00'
author: Andres Olarte
tags:
- jax-rs
- dropwizard
- JavaEE
- webservices
- rest
- tips
- java
modified_time: '2015-08-31T14:21:21.793-07:00'
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-4237356648039431869
blogger_orig_url: http://www.javaprocess.com/2015/08/a-simple-csv-messagebodywriter-for-jax.html
---
This is a very simple MessageBodyWriter that will allow you to output a List of objects as CSV from a JAX-RS webservice. Such services can be useful with frameworks such as [D3.js](http://d3js.org/). Jackson provides MessageBodyWriters for several formats, but it does not provide an out of the box solution for CSV. It does however offer several useful  classes to serialize objects into CSV. These are provided in the jackson-dataformat-csv artifact.
We can use those classes to create our own CSV `MessageBodyWritter` as shown below:

```java
package csv;


import com.fasterxml.jackson.dataformat.csv.CsvMapper;
import com.fasterxml.jackson.dataformat.csv.CsvSchema;

import javax.ws.rs.Produces;
import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.MultivaluedMap;
import javax.ws.rs.ext.MessageBodyWriter;
import javax.ws.rs.ext.Provider;
import java.io.IOException;
import java.io.OutputStream;
import java.lang.annotation.Annotation;
import java.lang.reflect.Type;
import java.util.List;

@Provider
@Produces("text/csv")
public class CSVMessageBodyWritter implements MessageBodyWriter {

    @Override
    public boolean isWriteable(Class type, Type genericType, Annotation[] annotations, MediaType mediaType) {
        boolean ret=List.class.isAssignableFrom(type);
        return ret;
    }

    @Override
    public long getSize(List data, Class aClass, Type type, Annotation[] annotations, MediaType mediaType) {
        return 0;
    }

    @Override
    public void writeTo(List data, Class aClass, Type type, Annotation[] annotations, MediaType mediaType, MultivaluedMap multivaluedMap, OutputStream outputStream) throws IOException, WebApplicationException {
        if (data!=null && data.size()>0) {
            CsvMapper mapper = new CsvMapper();
            Object o=data.get(0);
            CsvSchema schema = mapper.schemaFor(o.getClass()).withHeader();
            mapper.writer(schema).writeValue(outputStream,data);
        }


    }

}
```

To use our `MessageBodyWriter`, it must be registered. This can achieved in several ways, depending on your JAX-RS implementation. Normally Jersey and other JAX-RS implementations are configured to scan packages and look for resources. In such cases classed marked with `@Provider` will be registered automatically. In other cases, the registration will have to be done manually. For example in Dropwizard, you have to manually register the Writer at startup:

```java
    @Override
    public void run(MyConfiguration configuration,
                    Environment environment) {

        environment.jersey().register(new CSVMessageBodyWritter());
    }
```

Once registered, it becomes trivial to have a web service that outputs CSV. It's just a matter of annotating your webservice with the same media type as the `MessageBodyWritter`: `@Produces("text/csv")` in this case.


```java
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import java.util.List;

@Path("/status")
public class StatusResource {
    @GET
    @Produces("text/csv")
    public List<Data> getData() {
        List<Data> data= service.getStatus();
        return data;
    }
}
```

Our data class is just a normal POJO:

```java 
public class Data {
    private String date;
    private Integer minimum;
    private Integer maximum;
    private Integer average;

    public Data() {

    }

    //Getters and setters as needed
}
```

The output will look like this:

```
average,date,maximum,minimum
90,3/1856,125,0
60,2/1856,115,16
60,4/1856,115,16
```
        
This is a very simple implementation, but should be a good starting point. It's worth nothing that CSV is inherently limited, and can't easily represent hierarchical object graphs. Therefore you might need flatten your data before exporting to CSV. If you need to ingest a CSV in a webservice, you can follow a similar approach to create a `MessageBodyReader` that will create an object from a CSV stream.