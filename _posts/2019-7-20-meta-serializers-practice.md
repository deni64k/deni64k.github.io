---
layout: post
title: "Serialization with no efforts: Practice"
categories: [c++]
tags:
  serialization
  meta-programming
  concepts
  reflection
  metaclasses
---

In my [previous post]({% post_url 2019-01-26-meta-serializers %}), I described a way to deal with serialization using meta-programming.

Now, I want to show an example. Notably, we will see a Bitcoin protocol implementation without unnecessary boilerplate.

## Bitcoin 101

You have probably already heard about Bitcoin, a cryptocurrency that relies on a network of peers.

Here, we will focus on the protocol peers use to communicate with one another.
The full description of the protocol is available here: [Protocol documentation](https://en.bitcoin.it/wiki/Protocol_documentation), which is kind of sparse and hard to follow. In a few words, we can say it:

* is binary,
* consists of a bunch of commands,
* and every command consists of fields of particular types:
  * `uintN_t`,
  * `char[N]`, 
  * IPs, and
  * variable-length strings/integers.

Once a peer, Bitcoin client application, opens an outbound connection, it advertises its version with a command `version`. If the other peer accepts it, it replies with `verack` and sends its version.

We will focus on implementing a simple command [`version`](https://en.bitcoin.it/wiki/Protocol_documentation#version), and a more complicated one, [`tx`](https://en.bitcoin.it/wiki/Protocol_documentation#tx) - it has conditional fields.

All the time, I'll be referring to my implementation available here: [](https://github.com/deni64k/btc).

## Implementation

### Basic types

Before implementing commands, we should define our basic types forming the Bitcoin commands. For fundamental types such as `uint32_t`, `char[16]`, and variable-length strings, it is easy. We don't have to do anything besides the transport layer (about which I will tell later.)

For the types with special encoding, e.g., variable-length integers, we give it a separate type, [`var_int`](https://github.com/deni64k/btc/blob/master/src/common/types.hxx#L15):

```c++
struct var_int {
  std::uint64_t num;
  constexpr operator std::uint64_t () const noexcept { return num; }
};
```

This allows us to distinguish `var_int` from other integers in template specializations.

Finally, for a complex type such as IP address, there has to be created a type [`net_addr`](https://github.com/deni64k/btc/blob/master/src/common/types.hxx#L34):

```c++
using addr_t = std::array<std::uint8_t, 16>;

struct net_addr {
  // The Time (version >= 31402). Not present in version message.
  std::uint32_t time;
  // Same service(s) listed in version
  std::uint64_t services;
  // IPv6 address. Network byte order.
  // (12 bytes 00 00 00 00 00 00 00 00 00 00 FF FF, followed by the 4 bytes of the IPv4 address).
  addr_t addr;
  // Port number, network byte order
  be_uint16_t port;

  SERDES(time, services, addr, port)
};
```

Note, at the last line macro `SERDES`. It has the same purpose as `EXPOSE_MEMEBERS`, but called as a short version of 'Serialization and Deserialization.'

### Commands

Now, let's have a look at commands.

#### version

All commands are declared in [`proto/commands.hxx`](https://github.com/deni64k/btc/blob/master/src/proto/commands.hxx).

In particular, here is `version`:

```c++
struct version {
  enum: std::uint64_t {
    NODE_NETWORK = 1,
    NODE_GETUTXO = 2,
    NODE_BLOOM = 4,
    NODE_WITNESS = 8,
    NODE_NETWORK_LIMITED = 1024
  };

  // Identifies protocol version being used by the node
  std::uint32_t version;
  // Bitfield of features to be enabled for this connection
  std::uint64_t services;
  // Standard UNIX timestamp in seconds
  std::int64_t timestamp;
  // The network address of the node receiving this message
  net_addr_version addr_recv;
  // The network address of the node emitting this message
  net_addr_version addr_from;
  // Node random nonce, randomly generated every time a version packet is sent.
  // This nonce is used to detect connections to self.
  std::uint64_t nonce;
  // User Agent (0x00 if string is 0 bytes long)
  //var_str user_agent;
  var_str user_agent;
  // The last block received by the emitting node
  std::uint32_t start_height;
  // Whether the remote peer should announce relayed transactions or not, see BIP 0037
  std::uint8_t relay;

  SERDES(version, services, timestamp, addr_recv, addr_from, nonce,
         user_agent, start_height, relay)
};
```

It is a pretty straightforward list of command fields, and their exposure via `SERDES` that allows unified access to them.

Note, we have to use a different variation of `net_addr`, `net_addr_version` that doesn't have `time` field. Such is the protocol. (shrugs)

#### tx

Now, let's tackle something more complicated, [`tx`](https://github.com/deni64k/btc/blob/master/src/proto/commands.hxx#L209):

```c++
struct tx {
  // Transaction data format version (note, this is signed)
  std::int32_t version;
  // If present, always 0001, and indicates the presence of witness data
  // std::array<std::uint8_t, 2> flag;
  // A list of 1 or more transaction inputs or sources for coins
  std::vector<tx_in> txs_in;
  // A list of 1 or more transaction outputs or destinations for coins
  std::vector<tx_out> txs_out;
  // A list of witnesses, one for each input; omitted if flag is omitted above
  std::vector<tx_witness> tx_witnesses;
  // The block number or timestamp at which this transaction is unlocked
  // =  0          Not locked
  // <  500000000  Block number at which this transaction is unlocked
  // >= 500000000  UNIX timestamp at which this transaction is unlocked
  // If all TxIn inputs have final (0xffffffff) sequence numbers then lock_time is
  // irrelevant. Otherwise, the transaction may not be added to a block until after
  // lock_time (see NLockTime).
  std::uint32_t lock_time;

  SERDES(version, txs_in, txs_out, tx_witnesses, lock_time)

  auto total_value() const {
    std::uint64_t value = 0;
    for (auto&& tx : txs_out)
      value += tx.value;
    return value;
  }
};
```

As you may notice, the comments tell that the field `flag` may or not be present depending if a transaction contains witnesses. Obviously, a naive way of traversing blindly over all fields will not work out here.

To get it working, we will have to write a template specialization, which we will see soon.

### Transport Layer

As we have got our commands declared, we should think about a transport layer implementation.

Since the Bitcoin network works using TCP/IP stack, we can implement abstractions over a file descriptor and read/write operations. For this purposes, we declare a template type [`io_ops`](https://github.com/deni64k/btc/blob/master/src/io/ops.hxx#L8) with a bunch of specializations.

Here is the primary template:

```c++
template <typename base_ops, typename T>
struct io_ops {
  static void read(base_ops& io, T& o) {
    using o_type = std::remove_cvref_t<T>;
    io.read_impl(reinterpret_cast<char *>(&o), sizeof(o_type));
  }
  static void write(base_ops& io, T const& o) {
    using o_type = std::remove_cvref_t<T>;
    io.write_impl(reinterpret_cast<char const*>(&o), sizeof(o_type));
  }
};
```

There below goes a few specializations for arrays, big endian uint16_t (the protocol mainly relies on little endian), variable-length integers, and so on.

Note, there is a special case with [`tx`](https://github.com/deni64k/btc/blob/master/src/io/ops.hxx#L136) command:

```c++
template <typename base_ops>
struct io_ops<base_ops, proto::tx> {
  static void read(base_ops& io, proto::tx& o) {
    io_ops<base_ops, decltype(o.version)>::read(io, o.version);
    bool has_witnesses = false;
    var_int len;
    // NB: The number of transactions is never zero.
    io_ops<base_ops, decltype(len)>::read(io, len);
    // NB: Determine if we read a zero.
    if (len.num == 0) {
      // NB: If so, this command contains two bytes of 00 11, and has witnesses.
      has_witnesses = true;
      char c;
      io_ops<base_ops, decltype(c)>::read(io, c);
      // NB: Read the number of transactions.
      io_ops<base_ops, decltype(len)>::read(io, len);
    }
    for (unsigned i = 0; i < len.num; ++i) {
      proto::tx_in tx;
      io_ops<base_ops, decltype(tx)>::read(io, tx);
      o.txs_in.push_back(std::move(tx));
    }
    io_ops<base_ops, decltype(o.txs_out)>::read(io, o.txs_out);
    // NB: Conditionally read withnesses.
    if (has_witnesses) {
      io_ops<base_ops, decltype(o.tx_witnesses)>::read(io, o.tx_witnesses);
    }
    io_ops<base_ops, decltype(o.lock_time)>::read(io, o.lock_time);
  }

  static void write(base_ops& io, proto::tx const& o) {
    bool has_witnesses = !o.tx_witnesses.empty();
    io_ops<base_ops, decltype(o.version)>::write(io, o.version);
    // NB: Conditionally write flag field
    if (has_witnesses) {
      std::array<char, 2> flag = {0, 1};
      io_ops<base_ops, decltype(flag)>::write(io, flag);
    }
    io_ops<base_ops, decltype(o.txs_in)>::write(io, o.txs_in);
    io_ops<base_ops, decltype(o.txs_out)>::write(io, o.txs_out);
    // NB: Conditionally write withnesses
    if (has_witnesses)
      io_ops<base_ops, decltype(o.tx_witnesses)>::write(io, o.tx_witnesses);
    io_ops<base_ops, decltype(o.lock_time)>::write(io, o.lock_time);
  }
};
```

The thing is a command may or may not contain witnesses. If it contains, then there will be
two bytes, `00 01`, prior to the number of transactions. Since their number is never zero and
stored as an encoded integer, i.e., zero is a single byte, `00`, this can be used as a condition.
That is, if you read a zero as a number of transaction, it means you got the first byte
of the `00 01`. And, importantly, such command contains witnesses. Then you skip the second byte
and read the actual number of transaction. In the code, you can see where boolean `has_witnesses`
is assigned to `true`.

(I don't know why they made it complicated, but it's a good example for handling special cases.)

There below you will find a [specialization](https://github.com/deni64k/btc/blob/master/src/io/ops.hxx#L176)
for types satisfying `SerDes` concepts, i.e., types containing exposed member variables.
I leave digesting it to the reader as a good exercise in meta-programming.

### Actual IO

You may wonder where the actual IO operations are? And that is a good question.
In my implementation, there are moved into two classes:
* [`socket_ops`](https://github.com/deni64k/btc/blob/master/src/io/ops.hxx#L199) — implements operations over a file descriptor and throws an exception in case of failure, and
* [`ostream_ops`](https://github.com/deni64k/btc/blob/master/src/io/ops.hxx#L234) — similarly, implements operations over an instance of `std::ostream`.

## Usage

Let's see how we can use it now. First, we need helper functions hiding all the ugliness and complexity:

```c++
template <SerDes T>
void from_socket(int sock, T& payload) {
  socket_ops ops(sock);
  ops.read(payload);
}

template <SerDes T>
void to_socket(int sock, T const& payload) {
  socket_ops ops(sock);
  ops.write(payload);
}

template <SerDes T>
std::ostream& operator << (std::ostream& os, T const& payload) {
  ostream_ops ops(os);
  ops.write(payload);
  return os;
}
```

I admit, having a socket as a raw integer is not a good idea, but let's leave it as is for simplicity.

Then, wherever we want to send a `version`, we instantiate it and call to `to_socket`:

```c++
proto::version payload = { ...long list of values... };
// Every command has a header.
proto::header hdr = make_header("version", payload);

// Send it to a peer.
to_socket(peer, hdr);
to_socket(peer, payload);
```

Now, we may want to read the answer, `verack`, and print it (I assume you use `od -c` or `hexdump`,
otherwise you will see plain binary data in your console output.)

```c++
// Read a header.
proto::header hdr;
from_socket(peer, hdr);

// Ensure it is, indeed, verack.
assert(hdr.command_name == "verack");

// Read the command verack.
proto::verack payload;
from_socket(peer, payload);

// Print the verack to the console.
std::cerr << payload;
```

## That's All, Folks!
