pytsdl
======

**pytsdl** is a TSDL (Trace Stream Description Language) parser
implemented entirely in Python 3. TSDL is the language at the heart of
[CTF](http://git.efficios.com/?p=ctf.git;a=blob_plain;f=common-trace-format-specification.txt;hb=master),
the Common Trace Format.

pytsdl is able to parse a valid TSDL string and return either the
raw AST or a more useful object model of the document.

Although there are some limitations (see below), pytsdl successfully
parses a TSDL metadata file generated by [LTTng](http://lttng.org/).


installing
----------

Make sure you have `pip` for Python 3. On Ubuntu, it is called `pip3`:

    sudo apt-get install python3-pip

Install pyPEG2 first:

    sudo pip3 install pyPEG2

Install pytsdl:

    sudo pip3 install pytsdl


using
-----

Here are a few examples.

For all the following examples, `tsdl` is assumed to be a variable
containing a TSDL string and the prologue is:

    import pytsdl.parser

    parser = pytsdl.parser.Parser()

We are working with this TSDL document:

    /* CTF 1.8 */
    typealias integer { size = 8; align = 8; signed = false; } := uint8_t;
    typealias integer { size = 8; align = 8; signed = false; } := uint8_t;
    typealias integer { size = 16; align = 8; signed = false; } := uint16_t;
    typealias integer { size = 32; align = 8; signed = false; } := uint32_t;
    typealias integer { size = 64; align = 8; signed = false; } := uint64_t;
    typealias integer { size = 64; align = 8; signed = false; } := unsigned long;
    typealias integer { size = 5; align = 1; signed = false; } := uint5_t;
    typealias integer { size = 27; align = 1; signed = false; base = x; } := uint27_t;

    trace {
        major = 1;
        minor = 8;
        byte_order = be;
        uuid = "fa3cf4f6-9abd-dd42-b220-4d2b440b10e4";
        packet.header := struct {
            uint32_t magic;
            uint8_t  uuid[16];
            uint32_t stream_id;
        };
    };

    env {
        hostname = "archeepp";
        domain = "kernel";
        sysname = "Linux";
        kernel_release = "3.17.0-rc3-ARCH-lttng-00083-g57b252f-dirty";
        kernel_version = "#2 SMP PREEMPT Thu Sep 4 18:57:15 EDT 2014";
        tracer_name = "lttng-modules";
        tracer_major = 2;
        tracer_minor = 5;
        tracer_patchlevel = 0;
    };

    clock {
        name = monotonic;
        uuid = "8ca2ea5b-9331-430c-b2bc-414a9989c5f5";
        description = "Monotonic Clock";
        freq = 1000000000; /* Frequency, in Hz */
        /* clock value offset from Epoch is: offset * (1/freq) */
        offset = 1410027325724524018;
        offset_s = 29387928332;
        absolute = FALSE;
    };

    typealias integer {
        size = 27; align = 1; signed = false;
        map = clock.monotonic.value;
    } := uint27_clock_monotonic_t;

    typealias integer {
        size = 32; align = 8; signed = false;
        map = clock.monotonic.value;
    } := uint32_clock_monotonic_t;

    typealias integer {
        size = 64; align = 8; signed = false;
        map = clock.monotonic.value;
    } := uint64_clock_monotonic_t;

    struct packet_context {
        uint64_clock_monotonic_t timestamp_begin;
        uint64_clock_monotonic_t timestamp_end;
        uint64_t content_size;
        uint64_t packet_size;
        unsigned long events_discarded;
        uint32_t cpu_id;
    };

    struct event_header_compact {
        enum : uint5_t { compact = 0 ... 30, extended = 31 } id;
        variant <id> {
            struct {
                uint27_clock_monotonic_t timestamp;
            } compact;
            struct {
                uint32_t id;
                uint64_clock_monotonic_t timestamp;
            } extended;
        } v;
    } align(8);

    stream {
        id = 0;
        event.header := struct event_header_compact;
    };

    stream {
        id = 1;
        event.header := string;
        event.context := integer {
            align = 8;
            size = 5;
            encoding = UTF8;
        };
    };

    event {
        name = "simple_event";
        id = 0;
        stream_id = 0;
        fields := integer {
            size = 12;
        };
    };

    event {
        name = "other event";
        id = 23;
        stream_id = 0;
        fields := struct {
            string a;
            uint16_t b;
        } align(64);
    };

    event {
        name = "some_event";
        id = 0;
        stream_id = 1;

        variant named_variant {
            uint32_t ZERO;
            string {encoding = ASCII;} ONE;
            struct {
                unsigned long field[10];
            } align(16) ELEVEN;
        };

        fields := struct {
            struct a {
                unsigned long a;
                unsigned long b[23];
            } _some_field;

            typealias enum : unsigned long {
                ZERO,
                ONE,
                TWO,
                "the TEN" = 10,
                ELEVEN,
                "SOME RANGE" = 30...152,
            } := my_enum;

            struct a _field;
            struct a _field2[stream.event.header.id][150];
            my_enum _state;
            variant named_variant <_state> _yeah;
        };
    };



### object model

Once the TSDL document is parsed, querying the model is easy. Streams,
events, the environment, enumerations and compound types are all
subscriptable.

    >>> doc = parser.parse(tsdl)
    >>> doc.trace.major
    1

    >>> doc.trace.byte_order
    <ByteOrder.BE: 2>

    >>> doc.trace.uuid
    UUID('fa3cf4f6-9abd-dd42-b220-4d2b440b10e4')

    >>> doc.trace.packet_header
    <pytsdl.tsdl.Struct at 0x7f6dc24516d8>

    >>> doc.trace.packet_header['magic'].size
    32

    >>> doc.trace.packet_header['uuid'].element.signed
    False

    >>> doc.trace.packet_header['uuid'].length
    16

    >>> doc.env['hostname']
    'archeepp'

    >>> doc.env['tracer_major']
    2

    >>> doc.clocks['monotonic'].uuid
    UUID('8ca2ea5b-9331-430c-b2bc-414a9989c5f5')

    >>> len(doc.clocks)
    1

    >>> doc.clocks['monotonic'].absolute
    False

    >>> doc.streams[1].id
    1

    >>> doc.streams[1].event_header
    <pytsdl.tsdl.String at 0x7f6dc2451978>

    >>> doc.streams[0].event_header['id']
    <pytsdl.tsdl.Enum at 0x7f6dc2451080>

    >>> doc.streams[0].event_header['id'].integer.size
    5

    >>> doc.streams[0].event_header['id']['compact']
    (0, 30)

    >>> doc.streams[0].event_header['id']['extended']
    (31, 31)

    >>> doc.streams[0].event_header['id'].label_of(15)
    'compact'

    >>> doc.streams[0].event_header['id'].label_of(31)
    'extended'

    >>> doc.streams[0].event_header['v']['extended']['id'].size
    32

    >>> doc.streams[0].event_header.align
    8

    >>> len(doc.streams[0].events)
    2

    >>> doc.streams[0].get_event(23) is doc.streams[0].get_event('other event')
    True

    >>> doc.streams[0].get_event(23).fields['b'].size
    16

    >>> doc.streams[0].get_event('simple_event').fields.size
    12

    >>> doc.streams[1].get_event(0).name
    'some_event'

    >>> doc.streams[1].get_event(0).fields['_state'].value_of('TWO')
    (2, 2)

    >>> doc.streams[1].get_event(0).fields['_state'].label_of(10)
    'the TEN'

    >>> doc.streams[1].get_event(0).fields['_state']['SOME RANGE']
    (30, 152)

    >>> doc.streams[1].get_event(0).fields['_yeah'].tag
    ['_state']

    >>> doc.streams[1].get_event(0).fields['_field2']
    <pytsdl.tsdl.Array at 0x7fd66b2714a8>

    >>> doc.streams[1].get_event(0).fields['_field2'].element
    <pytsdl.tsdl.Sequence at 0x7fd66b271898>

    >>> doc.streams[1].get_event(0).fields['_field2'].element.length
    ['stream', 'event', 'header', 'id']


### get the AST

Getting the raw AST is easy:

    >>> ast = parser.get_ast(tsdl)

The AST is a big tree of nodes found as is in the document. Types are
not resolved and the semantic is not verified. AST nodes are rendered
as XML when converted to strings:

    >>> print(ast)

outputs (after being formatted by `xmllint`):

    <?xml version="1.0"?>
    <top>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint8_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint8_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>16</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint16_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>32</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint32_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>64</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint64_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>64</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>unsigned long</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>5</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>1</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint5_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>27</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>1</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>base</id>
            <unary-expr>
              <postfix-expr>
                <id>x</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint27_t</id>
      </typealias>
      <trace>
        <value-assign>
          <id>major</id>
          <unary-expr>
            <const-number>1</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>minor</id>
          <unary-expr>
            <const-number>8</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>byte_order</id>
          <unary-expr>
            <postfix-expr>
              <id>be</id>
            </postfix-expr>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>uuid</id>
          <unary-expr>
            <literal-string>fa3cf4f6-9abd-dd42-b220-4d2b440b10e4</literal-string>
          </unary-expr>
        </value-assign>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>packet</id>
              <dot/>
              <id>header</id>
            </postfix-expr>
          </unary-expr>
          <struct-full>
            <id-field>
              <id>uint32_t</id>
              <decl>
                <id>magic</id>
              </decl>
            </id-field>
            <id-field>
              <id>uint8_t</id>
              <decl>
                <id>uuid</id>
                <subscript-expr>
                  <unary-expr>
                    <const-number>16</const-number>
                  </unary-expr>
                </subscript-expr>
              </decl>
            </id-field>
            <id-field>
              <id>uint32_t</id>
              <decl>
                <id>stream_id</id>
              </decl>
            </id-field>
          </struct-full>
        </type-assign>
      </trace>
      <env>
        <value-assign>
          <id>hostname</id>
          <unary-expr>
            <literal-string>archeepp</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>domain</id>
          <unary-expr>
            <literal-string>kernel</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>sysname</id>
          <unary-expr>
            <literal-string>Linux</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>kernel_release</id>
          <unary-expr>
            <literal-string>3.17.0-rc3-ARCH-lttng-00083-g57b252f-dirty</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>kernel_version</id>
          <unary-expr>
            <literal-string>#2 SMP PREEMPT Thu Sep 4 18:57:15 EDT 2014</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>tracer_name</id>
          <unary-expr>
            <literal-string>lttng-modules</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>tracer_major</id>
          <unary-expr>
            <const-number>2</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>tracer_minor</id>
          <unary-expr>
            <const-number>5</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>tracer_patchlevel</id>
          <unary-expr>
            <const-number>0</const-number>
          </unary-expr>
        </value-assign>
      </env>
      <clock>
        <value-assign>
          <id>name</id>
          <unary-expr>
            <postfix-expr>
              <id>monotonic</id>
            </postfix-expr>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>uuid</id>
          <unary-expr>
            <literal-string>8ca2ea5b-9331-430c-b2bc-414a9989c5f5</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>description</id>
          <unary-expr>
            <literal-string>Monotonic Clock</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>freq</id>
          <unary-expr>
            <const-number>1000000000</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>offset</id>
          <unary-expr>
            <const-number>1410027325724524018</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>offset_s</id>
          <unary-expr>
            <const-number>29387928332</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>absolute</id>
          <unary-expr>
            <postfix-expr>
              <id>FALSE</id>
            </postfix-expr>
          </unary-expr>
        </value-assign>
      </clock>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>27</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>1</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>map</id>
            <unary-expr>
              <postfix-expr>
                <id>clock</id>
                <dot/>
                <id>monotonic</id>
                <dot/>
                <id>value</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint27_clock_monotonic_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>32</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>map</id>
            <unary-expr>
              <postfix-expr>
                <id>clock</id>
                <dot/>
                <id>monotonic</id>
                <dot/>
                <id>value</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint32_clock_monotonic_t</id>
      </typealias>
      <typealias>
        <integer>
          <value-assign>
            <id>size</id>
            <unary-expr>
              <const-number>64</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>align</id>
            <unary-expr>
              <const-number>8</const-number>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>signed</id>
            <unary-expr>
              <postfix-expr>
                <id>false</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
          <value-assign>
            <id>map</id>
            <unary-expr>
              <postfix-expr>
                <id>clock</id>
                <dot/>
                <id>monotonic</id>
                <dot/>
                <id>value</id>
              </postfix-expr>
            </unary-expr>
          </value-assign>
        </integer>
        <id>uint64_clock_monotonic_t</id>
      </typealias>
      <struct-full>
        <id>packet_context</id>
        <id-field>
          <id>uint64_clock_monotonic_t</id>
          <decl>
            <id>timestamp_begin</id>
          </decl>
        </id-field>
        <id-field>
          <id>uint64_clock_monotonic_t</id>
          <decl>
            <id>timestamp_end</id>
          </decl>
        </id-field>
        <id-field>
          <id>uint64_t</id>
          <decl>
            <id>content_size</id>
          </decl>
        </id-field>
        <id-field>
          <id>uint64_t</id>
          <decl>
            <id>packet_size</id>
          </decl>
        </id-field>
        <id-field>
          <id>unsigned long</id>
          <decl>
            <id>events_discarded</id>
          </decl>
        </id-field>
        <id-field>
          <id>uint32_t</id>
          <decl>
            <id>cpu_id</id>
          </decl>
        </id-field>
      </struct-full>
      <struct-full>
        <id>event_header_compact</id>
        <type-field>
          <enum>
            <id>uint5_t</id>
            <enumerators>
              <enum-range>
                <id>compact</id>
                <const-int-range>
                  <const-number>0</const-number>
                  <const-number>30</const-number>
                </const-int-range>
              </enum-range>
              <enum-value>
                <id>extended</id>
                <const-int>31</const-int>
              </enum-value>
            </enumerators>
          </enum>
          <decl>
            <id>id</id>
          </decl>
        </type-field>
        <type-field>
          <variant-full>
            <tag>
              <unary-expr>
                <postfix-expr>
                  <id>id</id>
                </postfix-expr>
              </unary-expr>
            </tag>
            <type-field>
              <struct-full>
                <id-field>
                  <id>uint27_clock_monotonic_t</id>
                  <decl>
                    <id>timestamp</id>
                  </decl>
                </id-field>
              </struct-full>
              <decl>
                <id>compact</id>
              </decl>
            </type-field>
            <type-field>
              <struct-full>
                <id-field>
                  <id>uint32_t</id>
                  <decl>
                    <id>id</id>
                  </decl>
                </id-field>
                <id-field>
                  <id>uint64_clock_monotonic_t</id>
                  <decl>
                    <id>timestamp</id>
                  </decl>
                </id-field>
              </struct-full>
              <decl>
                <id>extended</id>
              </decl>
            </type-field>
          </variant-full>
          <decl>
            <id>v</id>
          </decl>
        </type-field>
        <struct-align>
          <const-int>8</const-int>
        </struct-align>
      </struct-full>
      <stream>
        <value-assign>
          <id>id</id>
          <unary-expr>
            <const-number>0</const-number>
          </unary-expr>
        </value-assign>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>event</id>
              <dot/>
              <id>header</id>
            </postfix-expr>
          </unary-expr>
          <struct-ref>
            <id>event_header_compact</id>
          </struct-ref>
        </type-assign>
      </stream>
      <stream>
        <value-assign>
          <id>id</id>
          <unary-expr>
            <const-number>1</const-number>
          </unary-expr>
        </value-assign>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>event</id>
              <dot/>
              <id>header</id>
            </postfix-expr>
          </unary-expr>
          <string/>
        </type-assign>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>event</id>
              <dot/>
              <id>context</id>
            </postfix-expr>
          </unary-expr>
          <integer>
            <value-assign>
              <id>align</id>
              <unary-expr>
                <const-number>8</const-number>
              </unary-expr>
            </value-assign>
            <value-assign>
              <id>size</id>
              <unary-expr>
                <const-number>5</const-number>
              </unary-expr>
            </value-assign>
            <value-assign>
              <id>encoding</id>
              <unary-expr>
                <postfix-expr>
                  <id>UTF8</id>
                </postfix-expr>
              </unary-expr>
            </value-assign>
          </integer>
        </type-assign>
      </stream>
      <event>
        <value-assign>
          <id>name</id>
          <unary-expr>
            <literal-string>simple_event</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>id</id>
          <unary-expr>
            <const-number>0</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>stream_id</id>
          <unary-expr>
            <const-number>0</const-number>
          </unary-expr>
        </value-assign>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>fields</id>
            </postfix-expr>
          </unary-expr>
          <integer>
            <value-assign>
              <id>size</id>
              <unary-expr>
                <const-number>12</const-number>
              </unary-expr>
            </value-assign>
          </integer>
        </type-assign>
      </event>
      <event>
        <value-assign>
          <id>name</id>
          <unary-expr>
            <literal-string>other event</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>id</id>
          <unary-expr>
            <const-number>23</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>stream_id</id>
          <unary-expr>
            <const-number>0</const-number>
          </unary-expr>
        </value-assign>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>fields</id>
            </postfix-expr>
          </unary-expr>
          <struct-full>
            <type-field>
              <string/>
              <decl>
                <id>a</id>
              </decl>
            </type-field>
            <id-field>
              <id>uint16_t</id>
              <decl>
                <id>b</id>
              </decl>
            </id-field>
            <struct-align>
              <const-int>64</const-int>
            </struct-align>
          </struct-full>
        </type-assign>
      </event>
      <event>
        <value-assign>
          <id>name</id>
          <unary-expr>
            <literal-string>some_event</literal-string>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>id</id>
          <unary-expr>
            <const-number>0</const-number>
          </unary-expr>
        </value-assign>
        <value-assign>
          <id>stream_id</id>
          <unary-expr>
            <const-number>1</const-number>
          </unary-expr>
        </value-assign>
        <variant-full>
          <id>named_variant</id>
          <id-field>
            <id>uint32_t</id>
            <decl>
              <id>ZERO</id>
            </decl>
          </id-field>
          <type-field>
            <string>
              <value-assign>
                <id>encoding</id>
                <unary-expr>
                  <postfix-expr>
                    <id>ASCII</id>
                  </postfix-expr>
                </unary-expr>
              </value-assign>
            </string>
            <decl>
              <id>ONE</id>
            </decl>
          </type-field>
          <type-field>
            <struct-full>
              <id-field>
                <id>unsigned long</id>
                <decl>
                  <id>field</id>
                  <subscript-expr>
                    <unary-expr>
                      <const-number>10</const-number>
                    </unary-expr>
                  </subscript-expr>
                </decl>
              </id-field>
              <struct-align>
                <const-int>16</const-int>
              </struct-align>
            </struct-full>
            <decl>
              <id>ELEVEN</id>
            </decl>
          </type-field>
        </variant-full>
        <type-assign>
          <unary-expr>
            <postfix-expr>
              <id>fields</id>
            </postfix-expr>
          </unary-expr>
          <struct-full>
            <type-field>
              <struct-full>
                <id>a</id>
                <id-field>
                  <id>unsigned long</id>
                  <decl>
                    <id>a</id>
                  </decl>
                </id-field>
                <id-field>
                  <id>unsigned long</id>
                  <decl>
                    <id>b</id>
                    <subscript-expr>
                      <unary-expr>
                        <const-number>23</const-number>
                      </unary-expr>
                    </subscript-expr>
                  </decl>
                </id-field>
              </struct-full>
              <decl>
                <id>_some_field</id>
              </decl>
            </type-field>
            <typealias>
              <enum>
                <id>unsigned long</id>
                <enumerators>
                  <id>ZERO</id>
                  <id>ONE</id>
                  <id>TWO</id>
                  <enum-value>
                    <literal-string>the TEN</literal-string>
                    <const-int>10</const-int>
                  </enum-value>
                  <id>ELEVEN</id>
                  <enum-range>
                    <literal-string>SOME RANGE</literal-string>
                    <const-int-range>
                      <const-number>30</const-number>
                      <const-number>152</const-number>
                    </const-int-range>
                  </enum-range>
                </enumerators>
              </enum>
              <id>my_enum</id>
            </typealias>
            <type-field>
              <struct-ref>
                <id>a</id>
              </struct-ref>
              <decl>
                <id>_field</id>
              </decl>
            </type-field>
            <type-field>
              <struct-ref>
                <id>a</id>
              </struct-ref>
              <decl>
                <id>_field2</id>
                <subscript-expr>
                  <unary-expr>
                    <postfix-expr>
                      <id>stream</id>
                      <dot/>
                      <id>event</id>
                      <dot/>
                      <id>header</id>
                      <dot/>
                      <id>id</id>
                    </postfix-expr>
                  </unary-expr>
                </subscript-expr>
                <subscript-expr>
                  <unary-expr>
                    <const-number>150</const-number>
                  </unary-expr>
                </subscript-expr>
              </decl>
            </type-field>
            <id-field>
              <id>my_enum</id>
              <decl>
                <id>_state</id>
              </decl>
            </id-field>
            <type-field>
              <variant-ref>
                <id>named_variant</id>
                <tag>
                  <unary-expr>
                    <postfix-expr>
                      <id>_state</id>
                    </postfix-expr>
                  </unary-expr>
                </tag>
              </variant-ref>
              <decl>
                <id>_yeah</id>
              </decl>
            </type-field>
          </struct-full>
        </type-assign>
      </event>
    </top>


limitations
-----------

Current limitations:

  * super slow when parsing a big document (pyPEG2 limitation)
  * details of parsing errors are not always useful (pyPEG2 limitation)
  * partial semantic validation
  * `typedef` is not supported (`typealias` is)
  * GNU/C bitfields are not supported
  * pointer syntax is not supported (`struct hello*`, `name->something`,
    etc.)
  * only the dot notation is supported for variant tags and sequence
    lengths
