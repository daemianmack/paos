# PAOS

> Lightweight and easy-to-use library to build SOAP clients from WSDL files.

# Intro

Welcome to PAOS! This is a lightweight clojure library that enables developers to build rich SOAP interfaces from WSDL files without bringing in extra complexity with Java interop and autogenerated Java classes. PAOS is a wrapper for the awesome [soap-ws](https://github.com/reficio/soap-ws).

# Rationale

The main goal for PAOS is to provide developers with a flexible and easy-to-use means of communicating with web services accessible via SOAP protocol. Typically, these services provide a WSDL document that describes all available sub-services with a list of possible answers in the form of a structured description of data types. If you have ever worked with such documents, you most likely know how complex and convoluted they can get. For Clojure, almost all currently available approaches are dependent on the generation of Java classes with the help of tools such as wsdl2java or axis2. With PAOS, you no longer have to bother with fine-tuning super flexible professional tools every time a random letter has changed in your WSDL.

# When should you use PAOS?

- Your project works with dynamically changing services
- Your project needs to be able to establish communication with new services on the fly
- You do not want to dive into the abyss of Java interop with often suboptimally generated classes
- You need to somehow customize the messages sent to the service

# What does PAOS support?

All that [soap-ws](https://github.com/reficio/soap-ws) supports. If not – please let me know.

# Installation

Because soap-ws depends on a list of external dependencies that are not published in clojars or maven, you need to add them to the project's repository list yourself.

## leiningen

    {...
     :repositories [["reficio" "http://repo.reficio.org/maven/"]
                    ["soapui" "http://www.soapui.org/repository/maven2"]
                    ["enonic" "http://repo.enonic.com/public/"]
                    ...]
     :dependencies [[io.xapix/paos "0.1.1"]
                    ...]
    }

## boot

    (set-env! :repositories #(conj % ["reficio" {:url "http://repo.reficio.org/maven/"}]
                                     ["soapui" {:url "http://www.soapui.org/repository/maven2"}]
                                     ["enonic" {:url "http://repo.enonic.com/public/"}])
              :dependencies #(conj % [io.xapix/paos "0.1.1"])

## deps.edn

    {...
     :mvn/repos {"reficio"  {:url "http://repo.reficio.org/maven/"}
                 "soapui"   {:url "http://www.soapui.org/repository/maven2"}
                 "enonic"   {:url "http://repo.enonic.com/public/"}
                 "central"  {:url "https://repo1.maven.org/maven2/"}
                 "clojars"  {:url "https://clojars.org/repo/"}
                 "sonatype" {:url "https://oss.sonatype.org/content/repositories/snapshots/"}}}
     :deps {io.xapix/paos {:mvn/version "0.1.1"}
            ...}
    ...
    }

# TL;DR

```clojure
;; clj-http not included, add it yourself
(require '[clj-http.client :as client])
(require '[paos.service :as service])

(let [soap-service (parse "http://www.thomas-bayer.com/axis2/services/BLZService?wsdl")
      srv          (get-in soap-service ["BLZServiceSOAP11Binding" :operations "getBank"])
      soap-url     (get-in soap-service ["BLZServiceSOAP11Binding" :url])
      soap-action  (service/soap-action srv)
      mapping      (service/request-mapping srv)
      context      (assoc-in mapping ["Envelope" "Body" "getBank" "blz" :__value] "28350000")
      body         (service/wrap-body srv context)
      parse-fn     (partial service/parse-response srv)]
  (-> soap-url
      (client/post {:content-type "text/xml"
                    :body         body
                    :headers      {"SOAPAction" soap-action}})
      :body
      parse-fn))
```

# Quick start guide

Require the `paos.wsdl` namespace:

```clojure
(require '[paos.wsdl :as wsdl])
```

And process your WSDL using the function `wsdl/parse`

```clojure

;; You can use external urls
(def soap-service (wsdl/parse "http://example.com/some/wsdl/file?wsdl"))

;; Or absolute file path
(def soap-service (wsdl/parse "/some/place/service.wsdl"))

;; Or just pass the content of WSDL file as a string
(def soap-service (wsdl/parse "<SomeServise>"))
```


All service bindings should be visible as top-level keywords with all operations inside.

```clojure

soap-service
;; => {"SomeServiceBinding" {:operations {"operation1" ...
;;                                        "operation2" ...}
;;                           :url "service url"}
;;     "AnotherServiceBinding" {:operations {"operation3" ...
;;                                           "operation4" ...}
;;                              :url "another service url"}}
```

Each operation is an object with implemented with paos.service/Service methods which you can use to get the data necessary for that service:

```clojure

(require '[paos.service :as service])

(let [srv (get-in soap-service ["SomeServiceBinding" "operation1"]]
      (service/request-mapping srv))
  ;; => {"Envelope" {"Headers" []
  ;;                 "Body" {"SomeWrapper" {"Value" {:__value nil
  ;;                                                 :__type "string"}}}}}
```

`service/request-mapping` should give you an example how a request should look like to be converted into payload xml. Places where you have to add some real values are marked as `{:__value nil}` with the data type expected by the service in `{:__type "string"}`

Some data types can have arrays of complex objects:

```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
      <xs:element name="msgBody">
        <xs:complexType>
          <xs:sequence>
            <xs:element maxOccurs="unbounded" ref="Contato"/>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
      <xs:element name="Contato">
        <xs:complexType>
          <xs:sequence>
            <xs:element ref="cdEndereco"/>
            <xs:element ref="cdBairro"/>
            <xs:element ref="email"/>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
      <xs:element name="cdEndereco" type="xs:integer"/>
      <xs:element name="cdBairro" type="xs:integer"/>
      <xs:element name="email" type="xs:string"/>
    </xs:schema>
```

This type definition should be converted into a Clojure data structure like the following:

```clojure
{"msgBody"
 {"Contatos"
  [{"Contato"
    {"cdEndereco"
     {:__value nil
      :__type "string"}
     "cdBairro"
     {:__value nil
      :__type "string"}
     "email"
     {:__value nil
      :__type "integer"}}}]}}
```

Here in `"msgBody"` you can find the `"Contatos"` keyword with an array of one element. That means that the payload's `"msgBody"` might contain zero or more occurrences of `"Contato"` object. Just attach as many as you want into it and all of them will be injected correctly into the request payload.

To build the actual XML payload you have to use `wrap-body` function from the service namespace:

```clojure
(require '[paos.service :as service])

(let [srv (get-in soap-service ["SomeServiceBinding" "operation1"]
                  mapping (service/request-mapping srv)
                  context (do-something-with-mapping mapping)]
      (service/wrap-body srv context))
  ;; => "<xml>...</xml>"
```

The result is just a string with all your data inside.

Now you can use your favorite http library to make the request to the service:

```clojure
(require '[clj-http.client :as client])
(require '[paos.service :as service])

(let [soap-url (:url soap-service)
      srv (get-in soap-service ["SomeServiceBinding" "operation1"]
                  soap-action (service/soap-action srv)
                  mapping (service/request-mapping srv)
                  context (do-something-with-mapping mapping)
                  body (service/wrap-body srv context)]
      (client/post soap-url {:content-type "text/plain"
                             :body body
                             :headers {"SoapAction" soap-action}}))
  ;; => {:status 200
  ;;     :body "<xml>...</xml>"
  ;;     ...}
```

To convert the XML from the response you can use parse function from your service:

```clojure
(require '[clj-http.client :as client])
(require '[paos.service :as service])

(let [soap-service (parse "http://www.thomas-bayer.com/axis2/services/BLZService?wsdl")
      srv          (get-in soap-service ["BLZServiceSOAP11Binding" :operations "getBank"])
      soap-url     (get-in soap-service ["BLZServiceSOAP11Binding" :url])
      soap-action  (service/soap-action srv)
      mapping      (service/request-mapping srv)
      context      (assoc-in mapping ["Envelope" "Body" "getBank" "blz" :__value] "28350000")
      body         (service/wrap-body srv context)
      parse-fn     (partial service/parse-response srv)]
  (-> soap-url
      (client/post {:content-type "text/xml"
                    :body         body
                    :headers      {"SOAPAction" soap-action}})
      :body
      parse-fn))
```

# Copyright and License

Copyright © 2018 [Xapix GmbH](https://www.xapix.io/), and contributors

Distributed under the Eclipse Public License, the same as Clojure.
