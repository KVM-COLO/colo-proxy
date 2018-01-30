## Glossary

* PVM - Primary VM, which provides services to clients.
* SVM - Secondary VM, a hot standby and replication of PVM.
* PN - Primary Node, the host which PVM runs on
* SN - Secondary Node, the host which SVM runs on

## Implementation
We archive our goal by extending nf_conntrack mechanism.

There're 4 kernel modules in colo-proxy:
### nf_conntrack_colo
In this module We add an nf_conntrack extension named 'colo':
```
static struct nf_ct_ext_type nf_ct_colo_extend __read_mostly = {
	.len = sizeof(struct nf_conn_colo),
	.move = nf_ct_colo_extend_move,
	.destroy = nf_ct_colo_extend_destroy,
	.align = __alignof__(struct nf_conn_colo),
	.id = NF_CT_EXT_COLO,
};
```
This extension hold essential states needed by colo-proxy. e.g. manage the node status, the tcp connection status.

### xt_PMYCOLO
This module is for PN. It do the following operations:
* Register a xt_target(cooperate with iptables) to initiate the PN node status, run a kernel thread to compare packets.
```
static struct xt_target colo_primary_tg_regs[] __read_mostly = {
	{
		.name = "PMYCOLO",
		.family = NFPROTO_UNSPEC,
		.target = colo_primary_tg,
		.checkentry = colo_primary_tg_check,
		.destroy = colo_primary_tg_destroy,
		.targetsize = sizeof(struct xt_colo_primary_info),
		.table = "mangle",
		.hooks = (1 << NF_INET_PRE_ROUTING),
		.me = THIS_MODULE,
	},
};

static int colo_primary_tg_check(const struct xt_tgchk_param *par) 
{
	/*
	 * Setup forward device, init primary node status, create kthread for
	 * packets comparison.
	 */
}
```

* Register a nf_queue_handler to enqueue packets sent by PVM.

```
static const struct nf_queue_handler coloqh = {
	.outfn = &colo_enqueue_packet,
};
```

* Register some nf hooks to enqueue packets sent by SVM.

```
static struct nf_hook_ops colo_primary_ops[] __read_mostly = {
	{
		.hook = colo_slaver_queue_hook,
		.owner = THIS_MODULE,
		.pf = NFPROTO_IPV4,
		.hooknum = NF_INET_PRE_ROUTING,
		.priority = NF_IP_PRI_RAW + 1,
	},
	{
		.hook = colo_slaver_queue_hook,
		.owner = THIS_MODULE,
		.pf = NFPROTO_IPV6,
		.hooknum = NF_INET_PRE_ROUTING,
		.priority = NF_IP_PRI_RAW + 1,
	},
	{
		.hook = colo_slaver_arp_hook,
		.owner = THIS_MODULE,
		.pf = NFPROTO_ARP,
		.hooknum = NF_ARP_IN,
		.priority = NF_IP_PRI_FILTER + 1,
	},
};
```

### xt_SECCOLO
This module is for SN. It do the following operations:

* Register a xt_target(cooperate with iptables) to initiate the SN node status.

```
static struct xt_target colo_secondary_tg_regs[] __read_mostly = {
	{
		.name = "SECCOLO",
		.family = NFPROTO_UNSPEC,
		.target = colo_secondary_tg,
		.checkentry = colo_secondary_tg_check,
		.destroy = colo_secondary_tg_destroy,
		.targetsize = sizeof(struct xt_colo_secondary_info),
		.table = "mangle",
		.hooks = (1 << NF_INET_PRE_ROUTING),
		.me = THIS_MODULE,
	},
};
```

* Register some nf hooks to track the initial seq number of the tcp connections on both PVM/SVM, and do the seq adjustment for SVM(by
using the existing nf_conntrack_seqadj module).

```
static struct nf_hook_ops colo_secondary_ops[] __read_mostly = {
	{
		.hook = colo_secondary_hook,
		.owner = THIS_MODULE,
		.pf = NFPROTO_IPV4,
		.hooknum = NF_INET_PRE_ROUTING,
		.priority = NF_IP_PRI_MANGLE + 1,
	},
	{
		.hook = colo_secondary_hook,
		.owner = THIS_MODULE,
		.pf = NFPROTO_IPV6,.hooknum = NF_INET_PRE_ROUTING,
		.priority = NF_IP_PRI_MANGLE + 1,
	},
};
```

### nfnetlink_colo

This module is for communication with the userspace tools like QEMU or libxl.

In this module, add a colo protocol to the existing nfnetlink mechanism.
```
static const struct nfnetlink_subsystem nfulnl_subsys = {
	.name = "colo",
	.subsys_id = NFNL_SUBSYS_COLO,
	.cb_count = NFCOLO_MSG_MAX,
	.cb = nfnl_colo_cb,
};

static const struct nfnl_callback nfnl_colo_cb[NFCOLO_MSG_MAX] = {
	[NFCOLO_KERNEL_NOTIFY] = { .call = NULL,
		.policy = NULL,
		.attr_count = 0, },
	[NFCOLO_DO_CHECKPOINT] = { .call = colo_do_checkpoint,
		.policy = nfnl_colo_policy,
		.attr_count = NFNL_COLO_MAX, },
	[NFCOLO_DO_FAILOVER] = { .call = colo_do_failover,
		.policy = nfnl_colo_policy,
		.attr_count = NFNL_COLO_MAX, },
	[NFCOLO_PROXY_INIT] = { .call = colo_init_proxy,
		.policy = nfnl_colo_policy,
		.attr_count = NFNL_COLO_MAX, },
	[NFCOLO_PROXY_RESET] = { .call = colo_reset_proxy,
		.policy = nfnl_colo_policy,
		.attr_count = NFNL_COLO_MAX,},
};
```
