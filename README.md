- [Google Summer of Code 2024 Report](#google-summer-of-code-2024-report)
- [Project Overview](#project-overview)
  - [Background](#background)
  - [Project Status](#project-status)
    - [xtable kernel extension](#xtable-kernel-extension)
    - [xtable userspace extension](#xtable-userspace-extension)
    - [netfilter kernel extension](#netfilter-kernel-extension)
    - [conntrack](#conntrack)
    - [netlink support](#netlink-support)
  - [Additional Contributions](#additional-contributions)
  - [Benchmarks](#benchmarks)
- [Experience](#experience)


## Google Summer of Code 2024 Report

I am Mohammad Shehar Yaar Tausif, a final year under-graduate student at Indian Institute of 
Technology, Kharagpur (IIT KGP). I was selected as a **Student Contributor** for Google Summer of Code 2024 
at LabLua Foundation for the large-sized project [Lunatik binding for Netfilter](https://summerofcode.withgoogle.com/programs/2024/projects/BIJAPZjf).

> The proposal is available [here](./gsoc-2024-proposal-lablua.pdf) and the acceptance letter is available [here](gsoc-2024-acceptance-letter.pdf)

## Project Overview

### Background

Lunatik is a framework for scripting the Linux kernel with [Lua](https://www.lua.org/).
It is composed by the Lua interpreter modified to run in the kernel;
a [device driver](https://github.com/luainkernel/lunatik/blob/master/driver.lua) (written in Lua =)) and a [command line tool](https://github.com/luainkernel/lunatik/blob/master/bin/lunatik)
to load and run scripts and manage runtime environments from the user space;
a [C API](https://github.com/luainkernel/lunatik/tree/master?tab=readme-ov-file#c-api) to load and run scripts and manage runtime environments from the kernel;
and [Lua APIs](https://github.com/luainkernel/lunatik/tree/master?tab=readme-ov-file#lua-api) for binding kernel facilities to Lua scripts. 

The goal of the Google Summer of Code 2024 project was to enable lunatik users to write netfilter kernel modules, xtables (legacy) kernel and corresponding userspace extensions entirely in lua.

### Project Status

#### xtable kernel extension

The first step towards the project was to create lunatik bindings for xtable `target` and `match` kernel functions. These functions are used to register hooks in various netfilter tables like `iptables`, `arptables` and `ebtables`. 

Support for the following kernel functions, data structures and definitions were added to lunatik :
```c
/*** include/linux/netfilter/x_tables.h ***/
/* functions */
int xt_register_target(struct xt_target *target);
void xt_unregister_target(struct xt_target *target);
int xt_register_match(struct xt_match *target);
void xt_unregister_match(struct xt_match *target);
static inline unsigned int xt_hooknum(const struct xt_action_param *par)

/* structs */
struct xt_action_param;
struct xt_mtchk_param;
struct xt_mtdtor_param;
struct xt_tgchk_param;
struct xt_tgdtor_param;
struct xt_match;
struct xt_target;

/*** include/uapi/linux/netfilter.h ***/
/* defs */
#define NF_DROP 0
#define NF_ACCEPT 1
#define NF_STOLEN 2
#define NF_QUEUE 3
#define NF_REPEAT 4
#define NF_STOP 5
#define NF_MAX_VERDICT NF_STOP

/* enums */
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,
	NF_INET_LOCAL_IN,
	NF_INET_FORWARD,
	NF_INET_LOCAL_OUT,
	NF_INET_POST_ROUTING,
	NF_INET_NUMHOOKS,
	NF_INET_INGRESS = NF_INET_NUMHOOKS,
};

enum {
	NFPROTO_UNSPEC =  0,
	NFPROTO_INET   =  1,
	NFPROTO_IPV4   =  2,
	NFPROTO_ARP    =  3,
	NFPROTO_NETDEV =  5,
	NFPROTO_BRIDGE =  7,
	NFPROTO_IPV6   = 10,
	NFPROTO_NUMPROTO,
};

/*** include/uapi/linux/netfilter/x_tables.h ***/
#define XT_CONTINUE
#define XT_RETURN
```

An example to block DNS requests based on given domain name was addded which used these added APIs and the lunatik README was also updated accordingly.

**Pull Requests :**
1. [luanetfilter skeleton with flags #111](https://github.com/luainkernel/lunatik/pull/111)
2. [add xtable opts parsing and basic registration #113](https://github.com/luainkernel/lunatik/pull/113)
3. [add xtable rcu and lua callback execution #122](https://github.com/luainkernel/lunatik/pull/122)
4. [add luaxtable API #151](https://github.com/luainkernel/lunatik/pull/151)
5. [fix dnsblock example, add hooks field #152](https://github.com/luainkernel/lunatik/pull/152)

#### xtable userspace extension

The next step was to add support for userspace xtable extension which interacts with the `iptables` program and is responsible for sharing data from the userspace to the kernel space.

Support for the following userspace functions were added to lunatik :
```c
/*** include/xtables.h (userspace) ***/
struct xtables_match;
struct xtables_target;

extern void xtables_register_match(struct xtables_match *me);
extern void xtables_register_target(struct xtables_target *me);

/*** include/uapi/linux/x_tables.h ***/
struct xt_entry_match;
struct xt_entry_target;
```

Additional contributions :
- A makefile to help generate the shared library(`.so`) and install it to the xtables directory(`/usr/lib/xtables`) was added.
- A shared header for user-space and kernel-space library to pass userdata from user-space to kernel-space.
- An example to perform DNS Doctoring (DNS answer modification based on subnet) was added.
- The README was updated to reflect the changes. 

**Pull Request** : [add xtable userspace library #153](https://github.com/luainkernel/lunatik/pull/153) (in-review)

#### netfilter kernel extension

Support for the newer netfilter kernel extension framework was also added to lunatik. The related functions, variables and definitions that were integrated are : 

```c
/*** include/uapi/linux/netfilter.h ***/
struct nf_hook_ops;
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);

enum nf_dev_hooks {
	NF_NETDEV_INGRESS,
	NF_NETDEV_EGRESS,
	NF_NETDEV_NUMHOOKS
};

/*** include/uapi/linux/netfilter_bridge.h ***/
#define NF_BR_PRE_ROUTING	0
#define NF_BR_LOCAL_IN		1
#define NF_BR_FORWARD		2
#define NF_BR_LOCAL_OUT		3
#define NF_BR_POST_ROUTING	4
#define NF_BR_BROUTING		5
#define NF_BR_NUMHOOKS		6

enum nf_br_hook_priorities {
	NF_BR_PRI_FIRST = INT_MIN,
	NF_BR_PRI_NAT_DST_BRIDGED = -300,
	NF_BR_PRI_FILTER_BRIDGED = -200,
	NF_BR_PRI_BRNF = 0,
	NF_BR_PRI_NAT_DST_OTHER = 100,
	NF_BR_PRI_FILTER_OTHER = 200,
	NF_BR_PRI_NAT_SRC = 300,
	NF_BR_PRI_LAST = INT_MAX,
};

/*** include/uapi/linux/netfilter_arp.h ***/
#define NF_ARP_IN	0
#define NF_ARP_OUT	1
#define NF_ARP_FORWARD	2

/*** include/uapi/linux/netfilter_ipv4.h ***/
enum nf_ip_hook_priorities {
	NF_IP_PRI_FIRST = INT_MIN,
	NF_IP_PRI_RAW_BEFORE_DEFRAG = -450,
	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
	NF_IP_PRI_RAW = -300,
	NF_IP_PRI_SELINUX_FIRST = -225,
	NF_IP_PRI_CONNTRACK = -200,
	NF_IP_PRI_MANGLE = -150,
	NF_IP_PRI_NAT_DST = -100,
	NF_IP_PRI_FILTER = 0,
	NF_IP_PRI_SECURITY = 50,
	NF_IP_PRI_NAT_SRC = 100,
	NF_IP_PRI_SELINUX_LAST = 225,
	NF_IP_PRI_CONNTRACK_HELPER = 300,
	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
	NF_IP_PRI_LAST = INT_MAX,
};
```

Additional contributions :
- README updated according to the API changes
- Examples of DNS blocking and DNS doctoring was added similar to xtable examples.

**Pull Request** : [netfilter API #154](https://github.com/luainkernel/lunatik/pull/154) (in-review)

#### conntrack

Implementing conntrack and NAT support to the lunatik library proved to be a very large addition to the codebase 
and after discussion with the mentor, it was decided to keep it as a medium-sized (175h) project for the next 
Google Summer of Code. However, I was tasked to document the required deliverables and the reference kernel code to 
make the integration possible.

- Gist link for the document : https://gist.github.com/sheharyaar/cbbd43037fa43fd391e52964347a2bc9
- Issue linked to conntrack support : https://github.com/luainkernel/lunatik/issues/165

#### netlink support

By the time Google Summer of Code started, the [luasocket](https://github.com/luainkernel/lunatik?tab=readme-ov-file#socketaf) API already had support for netluink sockets.

### Additional Contributions

Lunatik library had prior support for [luadata](https://github.com/luainkernel/lunatik?tab=readme-ov-file#data), [luarcu](https://github.com/luainkernel/lunatik?tab=readme-ov-file#rcu) and [lualinux](https://github.com/luainkernel/lunatik?tab=readme-ov-file#linux). Howerer, these required a few changes to add xtable and netfilter support.

The additional contributions to various lunatik APIs are :
- [add rm, install and mkdir variables to makefile #112](https://github.com/luainkernel/lunatik/pull/112)
- [add settable method to rcu API #118](https://github.com/luainkernel/lunatik/pull/118)
- [add lunatik_checkfield method #119](https://github.com/luainkernel/lunatik/pull/119)
- [add set,get (u)int8,16,24,32,64 support to luadata #134](https://github.com/luainkernel/lunatik/pull/134)
- [fix luaxdp BTF flags for kernel >= 6.9.0 #137](https://github.com/luainkernel/lunatik/pull/137)
- [add editable field to luadata #139](https://github.com/luainkernel/lunatik/pull/139)
- [update README for luadata fixed size getter/setters #140](https://github.com/luainkernel/lunatik/pull/140)
- [add endian conversion support to lualinux #141](https://github.com/luainkernel/lunatik/pull/141)
- [fix luadata_checkwritable cond #143](https://github.com/luainkernel/lunatik/pull/143)
- [add lualinux endian conversion docs #146](https://github.com/luainkernel/lunatik/pull/146)
- [fix luadata opt in call to luadata_new #148](https://github.com/luainkernel/lunatik/pull/148)
- [fix lunatik sleep check #149](https://github.com/luainkernel/lunatik/pull/149)
- [fix luadata setter, getter docs #150](https://github.com/luainkernel/lunatik/pull/150)

### Benchmarks

## Experience

/* TODO */