=====
jesse [![Build Status](https://secure.travis-ci.org/klarna/jesse.png)](http://travis-ci.org/klarna/jesse)
=====

jesse (JSon Schema Erlang) is an implementation of a json schema validator
for Erlang.

jesse implements [Draft 03] (http://tools.ietf.org/html/draft-zyp-json-schema-03) of
the specification. It supports almost all core schema definitions except:

* format
* $ref

Quick start
-----------

There are two ways of using jesse:

* to use jesse internal in-memory storage to keep all your schema definitions
  In this case jesse will look up a schema definition in its own storage,
  and then validate given json.
* it is also possible to provide jesse with schema definitions when jesse is called.

Examples
--------

    NOTE: jesse doesn't have any parsing functionality. It currently works with three
          formats: mochijson2, jiffy and jsx, so json needs to be parsed in advance,
          or you can specify a callback which jesse will use to parse json.

          In examples below and in jesse test suite jiffy parser is used.

* Use jesse's internal in-memory storage:

(parse json in advance)

```erlang
1> Schema = jiffy:decode(<<"{\"items\": {\"type\": \"integer\"}}">>).
{[{<<"items">>,{[{<<"type">>,<<"integer">>}]}}]}
2> jesse:add_schema(some_key, Schema).
ok
3> Json1 = jiffy:decode(<<"[1, 2, 3]">>).
[1,2,3]
4> jesse:validate(some_key, Json1).
{ok,[1,2,3]}
5> Json2 = jiffy:decode(<<"[1, \"x\"]">>).
[1,<<"x">>]
6> jesse:validate(some_key, Json2).
{error,[{data_invalid,{[{<<"type">>,<<"integer">>}]},
                      wrong_type,<<"x">>}]}
```

(using a callback)

```erlang
1> jesse:add_schema(some_key,
1>                  <<"{\"uniqueItems\": true}">>,
1>                  [{parser_fun, fun jiffy:decode/1}]).
ok
2> jesse:validate(some_key,
2>                <<"[1, 2]">>,
2>                [{parser_fun, fun jiffy:decode/1}]).
{ok,[1, 2]}
3> jesse:validate(some_key,
3>                <<"[{\"foo\": \"bar\"}, {\"foo\": \"bar\"}] ">>,
3>                [{parser_fun, fun jiffy:decode/1}]).
{error,[{data_invalid,{[{<<"uniqueItems">>,true}]},
                      {not_unique,{[{<<"foo">>,<<"bar">>}]}},
                      [{[{<<"foo">>,<<"bar">>}]},{[{<<"foo">>,<<"bar">>}]}]}]}
```

* Call jesse with schema definition in place (do not use internal storage)

(parse json in advance)

```erlang
1> Schema = jiffy:decode(<<"{\"pattern\": \"^a*$\"}">>).
{[{<<"pattern">>,<<"^a*$">>}]}
2> Json1 = jiffy:decode(<<"\"aaa\"">>).
<<"aaa">>
3> jesse:validate_with_schema(Schema, Json1).
{ok,<<"aaa">>}
4> Json2 = jiffy:decode(<<"\"abc\"">>).
<<"abc">>
5> jesse:validate_with_schema(Schema, Json2).
{error,[{data_invalid,{[{<<"pattern">>,<<"^a*$">>}]},
                      no_match,<<"abc">>}]}
```

(using a callback)

```erlang
1> Schema = <<"{\"patternProperties\": {\"f.*o\": {\"type\": \"integer\"}}}">>.
<<"{\"patternProperties\": {\"f.*o\": {\"type\": \"integer\"}}}">>
2> jesse:validate_with_schema(Schema,
2>                            <<"{\"foo\": 1, \"foooooo\" : 2}">>,
2>                            [{parser_fun, fun jiffy:decode/1}]).
{ok,{[{<<"foo">>,1},{<<"foooooo">>,2}]}}
3> jesse:validate_with_schema(Schema,
3>                            <<"{\"foo\": \"bar\", \"fooooo\": 2}">>,
3>                            [{parser_fun, fun jiffy:decode/1}]).
{error,[{data_invalid,{[{<<"type">>,<<"integer">>}]},
                      wrong_type,<<"bar">>}]}
```

* Since 0.4.0 it's possible to say jesse to collect errors, and not stop
  immediately when it finds an error in the given json:

```erlang
1> Schema = <<"{\"properties\": {\"a\": {\"type\": \"integer\"}, \"b\": {\"type\": \"string\"}, \"c\": {\"type\": \"boolean\"}}}">>.
<<"{\"properties\": {\"a\": {\"type\": \"integer\"}, \"b\": {\"type\": \"string\"}, \"c\": {\"type\": \"boolean\"}}}">>
2> jesse:validate_with_schema(Schema,
2>                            <<"{\"a\": 1, \"b\": \"b\", \"c\": true}">>,
2>                            [{parser_fun, fun jiffy:decode/1}]).
{ok,{[{<<"a">>,1},{<<"b">>,<<"b">>},{<<"c">>,true}]}}
```

now let's change the value of the field "b" to an integer

```erlang
3> jesse:validate_with_schema(Schema,
3>                            <<"{\"a\": 1, \"b\": 2, \"c\": true}">>,
3>                            [{parser_fun, fun jiffy:decode/1}]).
{error,[{data_invalid,{[{<<"type">>,<<"string">>}]},
                      wrong_type,2}]}
```

works as expected, but let's change the value of the field "c" as well

```erlang
4> jesse:validate_with_schema(Schema,
4>                            <<"{\"a\": 1, \"b\": 2, \"c\": 3}">>,
4>                            [{parser_fun, fun jiffy:decode/1}]).
{error,[{data_invalid,{[{<<"type">>,<<"string">>}]},
                      wrong_type,2}]}
```

still works as expected, jesse stops validating as soon as finds an error.
let's use the 'allowed_errors' option, and set it to 1

```erlang
5> jesse:validate_with_schema(Schema,
5>                            <<"{\"a\": 1, \"b\": 2, \"c\": 3}">>,
5>                            [{parser_fun, fun jiffy:decode/1},
5>                             {allowed_errors, 1}]).
{error,[{data_invalid,{[{<<"type">>,<<"boolean">>}]},
                      wrong_type,3},
        {data_invalid,{[{<<"type">>,<<"string">>}]},wrong_type,2}]}
```

now we got a list of two errors. let's now change the value of the field "a"
to a boolean

```erlang
6> jesse:validate_with_schema(Schema,
6>                            <<"{\"a\": true, \"b\": 2, \"c\": 3}">>,
6>                            [{parser_fun, fun jiffy:decode/1},
6>                             {allowed_errors, 1}]).
{error,[{data_invalid,{[{<<"type">>,<<"string">>}]},
                      wrong_type,2},
        {data_invalid,{[{<<"type">>,<<"integer">>}]},
                      wrong_type,true}]}
```

we stil got only two errors. let's try using 'infinity' as the argument
for the 'allowed_errors' option

```erlang
7> jesse:validate_with_schema(Schema,
7>                            <<"{\"a\": true, \"b\": 2, \"c\": 3}">>,
7>                            [{parser_fun, fun jiffy:decode/1},
7>                             {allowed_errors, infinity}]).
{error,[{data_invalid,{[{<<"type">>,<<"boolean">>}]},
                      wrong_type,3},
        {data_invalid,{[{<<"type">>,<<"string">>}]},wrong_type,2},
        {data_invalid,{[{<<"type">>,<<"integer">>}]},
                      wrong_type,true}]}
```

Caveats
-------
* pattern and patternProperty attributes:

  jesse uses standard erlang module `re` for regexp matching, therefore there could be
  some incompatible regular expressions in schemas you define.

  From erlang docs: "re's matching algorithms are currently based on the PCRE library,
  but not all of the PCRE library is interfaced"

  But most of common cases should work fine.

Contributing
------------

If you see something missing or incorrect, a pull request is most welcome!