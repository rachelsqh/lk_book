netfilter分析
-------------


netfilter 是一个用于包处理的框架，在正常的 Berkeley 套接字接口之外。 它有四个部分。 首先，每个协议都定义了“钩子”（IPv4 定义了 5 个），它们是数据包遍历该协议栈时明确定义的点。 在这些点中的每一个，协议都将使用数据包和挂钩号调用 netfilter 框架

换句话说，netfilter 是一个工具，它使您能够使用回调来解析、更改或使用数据包。

Netfilter 提供了一种称为 netfilter 钩子的东西，这是一种使用回调来过滤内核内部数据包的方法。 有 5 种不同的 netfilter 钩子： 








1. NF_IP_PER_ROUNTING — 当数据包到达机器时调用此钩子。
2. NF_IP_LOCAL_IN — 当数据包发往机器本身时调用此挂钩。
3. NF_IP_FORWARD — 当数据包发往另​​一个接口时调用此挂钩。
4. NF_IP_POST_ROUTING — 当数据包在返回线路和机器外部时调用。
5. NF_IP_LOCAL_OUT — 当一个数据包在本地创建并被发送出去时，这个钩子被调用。 

在内核中使用 Netfilter Hooks

要在内核中使用 netfilter 钩子，您需要创建一个钩子函数，并使用 nf_register_hook接收结构 nf_hooks_ops*或者 nf_register_net_hook接收 struct net*和 struct nf_hooks_ops. 您需要根据内核版本选择功能。 这 struct包含以下字段： 



1. hook — 一个指向函数的指针，一旦钩子被触发就会被调用。 此功能来自类型 nf_hookfn它在不同版本的内核中具有不同的签名。 我建议您根据您使用的内核版本搜索正确的签名（请参阅参考资料 [2]）。 确保此函数返回 NF_DROP（丢弃数据包）， NF_ACCEPT（让数据包继续它的旅程）或 NF_QUEUE（如果要将数据包排队到用户空间处理）。
2. hooknum — 钩子标识符之一（例如 NF_IP_POST_ROUTING）。
3. pf - 协议族标识符（例如 PF_INET对于 IPv4）。
4. priority — 钩子的优先级（如果系统中注册了其他钩子）。 此优先级可以是枚举中定义的优先级之一 nf_ip_hook_priorities，其定义在 netfilter_ipv4.h文件（例如 NF_IP_PRI_FIRST, NF_IP_PRI_RAW）。
5. 目前，您可以忽略 *dev和 *priv字段。
    只是为了好玩，我直接从 Linux 内核源代码文档中添加了一个引用——
    “ struct net_device — DEVICE 结构。
    实际上，这整个结构是一个很大的错误。 混合 I/O
    具有严格“高级”数据的数据，它必须知道
    INET 模块中使用的几乎所有数据结构。” 

代码示例

在本例中，我将向您展示一个简单的 LKM，它丢弃任何 UDP 数据包（除了发往端口 53 — DNS 的 UDP 数据包），并接受任何 TCP 数据包。 任何其他数据包都将被丢弃。 


/*****************************************************
 * This code was compiled and tested on Ubuntu 18.04.1
 * with kernel version 4.15.0
 *****************************************************/

#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <linux/udp.h>

static struct nf_hook_ops *nfho = NULL;

static unsigned int hfunc(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)
{
	struct iphdr *iph;
	struct udphdr *udph;
	if (!skb)
		return NF_ACCEPT;

	iph = ip_hdr(skb);
	if (iph->protocol == IPPROTO_UDP) {
		udph = udp_hdr(skb);
		if (ntohs(udph->dest) == 53) {
			return NF_ACCEPT;
		}
	}
	else if (iph->protocol == IPPROTO_TCP) {
		return NF_ACCEPT;
	}
	
	return NF_DROP;
}

static int __init LKM_init(void)
{
	nfho = (struct nf_hook_ops*)kcalloc(1, sizeof(struct nf_hook_ops), GFP_KERNEL);
	
	/* Initialize netfilter hook */
	nfho->hook 	= (nf_hookfn*)hfunc;		/* hook function */
	nfho->hooknum 	= NF_INET_PRE_ROUTING;		/* received packets */
	nfho->pf 	= PF_INET;			/* IPv4 */
	nfho->priority 	= NF_IP_PRI_FIRST;		/* max hook priority */
	
	nf_register_net_hook(&init_net, nfho);
}

static void __exit LKM_exit(void)
{
	nf_unregister_net_hook(&init_net, nfho);
	kfree(nfho);
}

module_init(LKM_init);
module_exit(LKM_exit);




代码概述

    第 21 行——检查是否 skb( sk_buf— 套接字缓冲区）为空，如果是，则让此数据包继续其路由。
    第 24 行——提取 IP 协议标头，以便我们以后可以使用它来获取有关数据包的更多详细信息。
    第 25 和 31 行——使用我们之前提取的 IP 标头，并检查该数据包使用了哪种协议。
    第 27 行 — 检查 UDP 标头的端口。
    第 48 和 53 行——注册和注销钩子。 

生成文件

要编译 LKM，您可以使用名为 Makefile，这是一个包含指令和设置的文件，稍后将由 make命令在 bash. 对于这段代码，我写了以下 Makefile： 










编译和测试

编译并插入 LKM bash： 





和 rmmod LKM删除它。

我强烈建议使用 Wireshark 或其他嗅探工具来查看该模块如何影响您机器上的网络流量。 


























netfilter头
^^^^^^^^^^^^

/* Function to register/unregister hook points. */
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);
int nf_register_net_hooks(struct net *net, const struct nf_hook_ops *reg,
			  unsigned int n);
void nf_unregister_net_hooks(struct net *net, const struct nf_hook_ops *reg,
			     unsigned int n);

/* Functions to register get/setsockopt ranges (non-inclusive).  You
   need to check permissions yourself! */
int nf_register_sockopt(struct nf_sockopt_ops *reg);
void nf_unregister_sockopt(struct nf_sockopt_ops *reg);


typedef unsigned int nf_hookfn(void *priv,
			       struct sk_buff *skb,
			       const struct nf_hook_state *state);
enum nf_hook_ops_type {
	NF_HOOK_OP_UNDEFINED,
	NF_HOOK_OP_NF_TABLES,
};

struct nf_hook_ops {
	/* User fills in from here down. */
	nf_hookfn		*hook;
	struct net_device	*dev;
	void			*priv;
	u8			pf;
	enum nf_hook_ops_type	hook_ops_type:8;
	unsigned int		hooknum;
	/* Hooks are ordered in ascending priority. */
	int			priority;
};









netfilter 范例
^^^^^^^^^^^^^^^^^
我们看注册场景：














导出符号
^^^^^^^^^^

core.c:EXPORT_SYMBOL_GPL(nf_ipv6_ops);
core.c:EXPORT_SYMBOL_GPL(nf_skb_duplicated);
core.c:EXPORT_SYMBOL(nf_hooks_needed);
core.c:EXPORT_SYMBOL_GPL(nf_hook_entries_insert_raw);
core.c:EXPORT_SYMBOL(nf_unregister_net_hook);
core.c:EXPORT_SYMBOL_GPL(nf_hook_entries_delete_raw);
core.c:EXPORT_SYMBOL(nf_register_net_hook);
core.c:EXPORT_SYMBOL(nf_register_net_hooks);
core.c:EXPORT_SYMBOL(nf_unregister_net_hooks);
core.c:EXPORT_SYMBOL(nf_hook_slow);
core.c:EXPORT_SYMBOL(nf_hook_slow_list);
core.c:EXPORT_SYMBOL_GPL(nfnl_ct_hook);
core.c:EXPORT_SYMBOL_GPL(nf_ct_hook);
core.c:EXPORT_SYMBOL(ip_ct_attach);
core.c:EXPORT_SYMBOL_GPL(nf_nat_hook);
core.c:EXPORT_SYMBOL(nf_ct_attach);
core.c:EXPORT_SYMBOL(nf_conntrack_destroy);
core.c:EXPORT_SYMBOL(nf_ct_get_tuple_skb);
core.c:EXPORT_SYMBOL_GPL(nf_ct_zone_dflt);
ipset/pfxlen.c:EXPORT_SYMBOL_GPL(ip_set_netmask_map);
ipset/pfxlen.c:EXPORT_SYMBOL_GPL(ip_set_hostmask_map);
ipset/pfxlen.c:EXPORT_SYMBOL_GPL(ip_set_range_to_cidr);
ipset/ip_set_getport.c:EXPORT_SYMBOL_GPL(ip_set_get_ip4_port);
ipset/ip_set_getport.c:EXPORT_SYMBOL_GPL(ip_set_get_ip6_port);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_type_register);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_type_unregister);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_alloc);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_free);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_get_ipaddr4);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_get_ipaddr6);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_init_comment);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_extensions);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_elem_len);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_get_extensions);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_put_extensions);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_match_extensions);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_test);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_add);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_del);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_get_byname);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_put_byindex);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_name_byindex);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_nfnl_get_byindex);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_nfnl_put);
ipset/ip_set_core.c:EXPORT_SYMBOL_GPL(ip_set_put_flags);
ipvs/ip_vs_app.c:EXPORT_SYMBOL(register_ip_vs_app);
ipvs/ip_vs_app.c:EXPORT_SYMBOL(unregister_ip_vs_app);
ipvs/ip_vs_app.c:EXPORT_SYMBOL(register_ip_vs_app_inc);
ipvs/ip_vs_conn.c:EXPORT_SYMBOL_GPL(ip_vs_conn_in_get_proto);
ipvs/ip_vs_conn.c:EXPORT_SYMBOL_GPL(ip_vs_conn_out_get_proto);
ipvs/ip_vs_sched.c:EXPORT_SYMBOL(ip_vs_scheduler_err);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(register_ip_vs_scheduler);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(unregister_ip_vs_scheduler);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_proto_name);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_conn_new);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_conn_in_get);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_conn_out_get);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_tcp_conn_listen);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_conn_put);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_get_debug_level);
ipvs/ip_vs_core.c:EXPORT_SYMBOL(ip_vs_new_conn_out);
ipvs/ip_vs_pe.c:EXPORT_SYMBOL_GPL(register_ip_vs_pe);
ipvs/ip_vs_pe.c:EXPORT_SYMBOL_GPL(unregister_ip_vs_pe);
ipvs/ip_vs_nfct.c:EXPORT_SYMBOL(ip_vs_nfct_expect_related);
ipvs/ip_vs_proto.c:EXPORT_SYMBOL(ip_vs_proto_get);
ipvs/ip_vs_proto.c:EXPORT_SYMBOL(ip_vs_proto_data_get);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_add);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_list_init);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_gc_list);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_count);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_init);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_cache_free);
nf_conncount.c:EXPORT_SYMBOL_GPL(nf_conncount_destroy);
nf_conntrack_amanda.c:EXPORT_SYMBOL_GPL(nf_nat_amanda_hook);
nf_conntrack_broadcast.c:EXPORT_SYMBOL_GPL(nf_conntrack_broadcast_help);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_locks);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_expect_lock);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_hash);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_lock);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_htable_size);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_max);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_get_tuplepr);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_invert_tuple);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_get_id);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_tmpl_alloc);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_tmpl_free);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_delete);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_find_get);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_hash_check_insert);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_acct_add);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(__nf_conntrack_confirm);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_tuple_taken);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_alloc);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_free);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_in);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_conntrack_alter_reply);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(__nf_ct_refresh_acct);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_kill_acct);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_port_tuple_to_nlattr);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_port_nla_policy);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_port_nlattr_to_tuple);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_port_nlattr_tuple_size);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_unconfirmed_destroy);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_iterate_cleanup_net);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_iterate_destroy);
nf_conntrack_core.c:EXPORT_SYMBOL_GPL(nf_ct_alloc_hashtable);
nf_conntrack_ecache.c:EXPORT_SYMBOL_GPL(nf_conntrack_eventmask_report);
nf_conntrack_ecache.c:EXPORT_SYMBOL_GPL(nf_ct_deliver_cached_events);
nf_conntrack_ecache.c:EXPORT_SYMBOL_GPL(nf_conntrack_register_notifier);
nf_conntrack_ecache.c:EXPORT_SYMBOL_GPL(nf_conntrack_unregister_notifier);
nf_conntrack_ecache.c:EXPORT_SYMBOL_GPL(nf_ct_expect_register_notifier);
nf_conntrack_ecache.c:EXPORT_SYMBOL_GPL(nf_ct_expect_unregister_notifier);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_hsize);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_hash);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_unlink_expect_report);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_remove_expect);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(__nf_ct_expect_find);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_find_get);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_remove_expectations);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_unexpect_related);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_alloc);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_init);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_put);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_related_report);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_iterate_destroy);
nf_conntrack_expect.c:EXPORT_SYMBOL_GPL(nf_ct_expect_iterate_net);
nf_conntrack_extend.c:EXPORT_SYMBOL(nf_ct_ext_add);
nf_conntrack_extend.c:EXPORT_SYMBOL_GPL(nf_ct_extend_register);
nf_conntrack_extend.c:EXPORT_SYMBOL_GPL(nf_ct_extend_unregister);
nf_conntrack_ftp.c:EXPORT_SYMBOL_GPL(nf_nat_ftp_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(get_h225_addr);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(set_h245_addr_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(set_h225_addr_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(set_sig_addr_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(set_ras_addr_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(nat_rtp_rtcp_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(nat_t120_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(nat_h245_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(nat_callforwarding_hook);
nf_conntrack_h323_main.c:EXPORT_SYMBOL_GPL(nat_q931_hook);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_hash);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_hsize);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(__nf_conntrack_helper_find);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_conntrack_helper_try_module_get);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_conntrack_helper_put);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_nat_helper_try_module_get);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_nat_helper_put);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_ext_add);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(__nf_ct_try_assign_helper);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_expectfn_register);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_expectfn_unregister);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_expectfn_find_by_name);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_expectfn_find_by_symbol);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_log);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_conntrack_helper_register);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_conntrack_helper_unregister);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_ct_helper_init);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_conntrack_helpers_register);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_conntrack_helpers_unregister);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_nat_helper_register);
nf_conntrack_helper.c:EXPORT_SYMBOL_GPL(nf_nat_helper_unregister);
nf_conntrack_irc.c:EXPORT_SYMBOL_GPL(nf_nat_irc_hook);
nf_conntrack_labels.c:EXPORT_SYMBOL_GPL(nf_connlabels_replace);
nf_conntrack_labels.c:EXPORT_SYMBOL_GPL(nf_connlabels_get);
nf_conntrack_labels.c:EXPORT_SYMBOL_GPL(nf_connlabels_put);
nf_conntrack_pptp.c:EXPORT_SYMBOL_GPL(nf_nat_pptp_hook_outbound);
nf_conntrack_pptp.c:EXPORT_SYMBOL_GPL(nf_nat_pptp_hook_inbound);
nf_conntrack_pptp.c:EXPORT_SYMBOL_GPL(nf_nat_pptp_hook_exp_gre);
nf_conntrack_pptp.c:EXPORT_SYMBOL_GPL(nf_nat_pptp_hook_expectfn);
nf_conntrack_pptp.c:EXPORT_SYMBOL(pptp_msg_name);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_l4proto_log_invalid);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_ct_l4proto_log_invalid);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_ct_l4proto_find);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_confirm);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_ct_netns_get);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_ct_netns_put);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_ct_bridge_register);
nf_conntrack_proto.c:EXPORT_SYMBOL_GPL(nf_ct_bridge_unregister);
nf_conntrack_proto_gre.c:EXPORT_SYMBOL_GPL(nf_ct_gre_keymap_add);
nf_conntrack_proto_gre.c:EXPORT_SYMBOL_GPL(nf_ct_gre_keymap_destroy);
nf_conntrack_seqadj.c:EXPORT_SYMBOL_GPL(nf_ct_seqadj_init);
nf_conntrack_seqadj.c:EXPORT_SYMBOL_GPL(nf_ct_seqadj_set);
nf_conntrack_seqadj.c:EXPORT_SYMBOL_GPL(nf_ct_tcp_seqadj_set);
nf_conntrack_seqadj.c:EXPORT_SYMBOL_GPL(nf_ct_seq_adjust);
nf_conntrack_seqadj.c:EXPORT_SYMBOL_GPL(nf_ct_seq_offset);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(nf_nat_sip_hooks);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(ct_sip_parse_request);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(ct_sip_get_header);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(ct_sip_parse_header_uri);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(ct_sip_parse_address_param);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(ct_sip_parse_numerical_param);
nf_conntrack_sip.c:EXPORT_SYMBOL_GPL(ct_sip_get_sdp_header);
nf_conntrack_snmp.c:EXPORT_SYMBOL_GPL(nf_nat_snmp_hook);
nf_conntrack_standalone.c:EXPORT_SYMBOL_GPL(print_tuple);
nf_conntrack_tftp.c:EXPORT_SYMBOL_GPL(nf_nat_tftp_hook);
nf_conntrack_timeout.c:EXPORT_SYMBOL_GPL(nf_ct_timeout_find_get_hook);
nf_conntrack_timeout.c:EXPORT_SYMBOL_GPL(nf_ct_timeout_put_hook);
nf_conntrack_timeout.c:EXPORT_SYMBOL_GPL(nf_ct_untimeout);
nf_conntrack_timeout.c:EXPORT_SYMBOL_GPL(nf_ct_set_timeout);
nf_conntrack_timeout.c:EXPORT_SYMBOL_GPL(nf_ct_destroy_timeout);
nf_dup_netdev.c:EXPORT_SYMBOL_GPL(nf_fwd_netdev_egress);
nf_dup_netdev.c:EXPORT_SYMBOL_GPL(nf_dup_netdev_egress);
nf_dup_netdev.c:EXPORT_SYMBOL_GPL(nft_fwd_dup_netdev_offload);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_alloc);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_route_init);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_free);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_add);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_refresh);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_teardown);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(flow_offload_lookup);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(nf_flow_snat_port);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(nf_flow_dnat_port);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(nf_flow_table_init);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(nf_flow_table_cleanup);
nf_flow_table_core.c:EXPORT_SYMBOL_GPL(nf_flow_table_free);
nf_flow_table_ip.c:EXPORT_SYMBOL_GPL(nf_flow_offload_ip_hook);
nf_flow_table_ip.c:EXPORT_SYMBOL_GPL(nf_flow_offload_ipv6_hook);
nf_flow_table_offload.c:EXPORT_SYMBOL_GPL(nf_flow_rule_route_ipv4);
nf_flow_table_offload.c:EXPORT_SYMBOL_GPL(nf_flow_rule_route_ipv6);
nf_flow_table_offload.c:EXPORT_SYMBOL_GPL(nf_flow_table_offload_setup);
nf_log.c:EXPORT_SYMBOL(sysctl_nf_log_all_netns);
nf_log.c:EXPORT_SYMBOL(nf_log_set);
nf_log.c:EXPORT_SYMBOL(nf_log_unset);
nf_log.c:EXPORT_SYMBOL(nf_log_register);
nf_log.c:EXPORT_SYMBOL(nf_log_unregister);
nf_log.c:EXPORT_SYMBOL(nf_log_bind_pf);
nf_log.c:EXPORT_SYMBOL(nf_log_unbind_pf);
nf_log.c:EXPORT_SYMBOL_GPL(nf_logger_request_module);
nf_log.c:EXPORT_SYMBOL_GPL(nf_logger_find_get);
nf_log.c:EXPORT_SYMBOL_GPL(nf_logger_put);
nf_log.c:EXPORT_SYMBOL(nf_log_packet);
nf_log.c:EXPORT_SYMBOL(nf_log_trace);
nf_log.c:EXPORT_SYMBOL_GPL(nf_log_buf_add);
nf_log.c:EXPORT_SYMBOL_GPL(nf_log_buf_open);
nf_log.c:EXPORT_SYMBOL_GPL(nf_log_buf_close);
nf_log_common.c:EXPORT_SYMBOL_GPL(nf_log_dump_udp_header);
nf_log_common.c:EXPORT_SYMBOL_GPL(nf_log_dump_tcp_header);
nf_log_common.c:EXPORT_SYMBOL_GPL(nf_log_dump_sk_uid_gid);
nf_log_common.c:EXPORT_SYMBOL_GPL(nf_log_dump_packet_common);
nf_log_common.c:EXPORT_SYMBOL_GPL(nf_log_dump_vlan);
nf_log_common.c:EXPORT_SYMBOL_GPL(nf_log_l2packet);
nf_nat_core.c:EXPORT_SYMBOL(nf_xfrm_me_harder);
nf_nat_core.c:EXPORT_SYMBOL_GPL(nf_ct_nat_ext_add);
nf_nat_core.c:EXPORT_SYMBOL(nf_nat_setup_info);
nf_nat_core.c:EXPORT_SYMBOL_GPL(nf_nat_alloc_null_binding);
nf_nat_core.c:EXPORT_SYMBOL_GPL(nf_nat_packet);
nf_nat_core.c:EXPORT_SYMBOL_GPL(nf_nat_inet_fn);
nf_nat_helper.c:EXPORT_SYMBOL(__nf_nat_mangle_tcp_packet);
nf_nat_helper.c:EXPORT_SYMBOL(nf_nat_mangle_udp_packet);
nf_nat_helper.c:EXPORT_SYMBOL(nf_nat_follow_master);
nf_nat_masquerade.c:EXPORT_SYMBOL_GPL(nf_nat_masquerade_ipv4);
nf_nat_masquerade.c:EXPORT_SYMBOL_GPL(nf_nat_masquerade_ipv6);
nf_nat_masquerade.c:EXPORT_SYMBOL_GPL(nf_nat_masquerade_inet_register_notifiers);
nf_nat_masquerade.c:EXPORT_SYMBOL_GPL(nf_nat_masquerade_inet_unregister_notifiers);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_icmp_reply_translation);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_ipv4_register_fn);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_ipv4_unregister_fn);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_icmpv6_reply_translation);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_ipv6_register_fn);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_ipv6_unregister_fn);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_inet_register_fn);
nf_nat_proto.c:EXPORT_SYMBOL_GPL(nf_nat_inet_unregister_fn);
nf_nat_redirect.c:EXPORT_SYMBOL_GPL(nf_nat_redirect_ipv4);
nf_nat_redirect.c:EXPORT_SYMBOL_GPL(nf_nat_redirect_ipv6);
nfnetlink_acct.c:EXPORT_SYMBOL_GPL(nfnl_acct_find_get);
nfnetlink_acct.c:EXPORT_SYMBOL_GPL(nfnl_acct_put);
nfnetlink_acct.c:EXPORT_SYMBOL_GPL(nfnl_acct_update);
nfnetlink_acct.c:EXPORT_SYMBOL_GPL(nfnl_acct_overquota);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnl_lock);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnl_unlock);
nfnetlink.c:EXPORT_SYMBOL_GPL(lockdep_nfnl_is_held);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnetlink_subsys_register);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnetlink_subsys_unregister);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnetlink_has_listeners);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnetlink_send);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnetlink_set_err);
nfnetlink.c:EXPORT_SYMBOL_GPL(nfnetlink_unicast);
nfnetlink_osf.c:EXPORT_SYMBOL_GPL(nf_osf_fingers);
nfnetlink_osf.c:EXPORT_SYMBOL_GPL(nf_osf_match);
nfnetlink_osf.c:EXPORT_SYMBOL_GPL(nf_osf_find);
nf_queue.c:EXPORT_SYMBOL(nf_register_queue_handler);
nf_queue.c:EXPORT_SYMBOL(nf_unregister_queue_handler);
nf_queue.c:EXPORT_SYMBOL_GPL(nf_queue_entry_free);
nf_queue.c:EXPORT_SYMBOL_GPL(nf_queue_entry_get_refs);
nf_queue.c:EXPORT_SYMBOL_GPL(nf_queue_nf_hook_drop);
nf_queue.c:EXPORT_SYMBOL_GPL(nf_queue);
nf_queue.c:EXPORT_SYMBOL(nf_reinject);
nf_sockopt.c:EXPORT_SYMBOL(nf_register_sockopt);
nf_sockopt.c:EXPORT_SYMBOL(nf_unregister_sockopt);
nf_sockopt.c:EXPORT_SYMBOL(nf_setsockopt);
nf_sockopt.c:EXPORT_SYMBOL(nf_getsockopt);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_net_id);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_parse_options);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_init_timestamp_cookie);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_send_client_synack);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_recv_client_ack);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(ipv4_synproxy_hook);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(nf_synproxy_ipv4_init);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(nf_synproxy_ipv4_fini);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_send_client_synack_ipv6);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(synproxy_recv_client_ack_ipv6);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(ipv6_synproxy_hook);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(nf_synproxy_ipv6_init);
nf_synproxy_core.c:EXPORT_SYMBOL_GPL(nf_synproxy_ipv6_fini);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_register_chain_type);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_unregister_chain_type);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_register_expr);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_unregister_expr);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_chain_validate);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_set_lookup_global);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nf_tables_bind_set);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nf_tables_deactivate_set);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nf_tables_destroy_set);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_set_elem_destroy);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_register_obj);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_unregister_obj);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_obj_lookup);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_obj_notify);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_register_flowtable_type);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_unregister_flowtable_type);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_flowtable_lookup);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nf_tables_deactivate_flowtable);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nf_tables_trans_destroy_flush_work);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_chain_validate_dependency);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_chain_validate_hooks);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_parse_u32_check);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_parse_register);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_dump_register);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_validate_register_load);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_validate_register_store);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_data_init);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_data_release);
nf_tables_api.c:EXPORT_SYMBOL_GPL(nft_data_dump);
nf_tables_api.c:EXPORT_SYMBOL_GPL(__nft_release_basechain);
nf_tables_core.c:EXPORT_SYMBOL_GPL(nft_do_chain);
nf_tables_trace.c:EXPORT_SYMBOL_GPL(nft_trace_enabled);
nft_fib.c:EXPORT_SYMBOL(nft_fib_policy);
nft_fib.c:EXPORT_SYMBOL_GPL(nft_fib_validate);
nft_fib.c:EXPORT_SYMBOL_GPL(nft_fib_init);
nft_fib.c:EXPORT_SYMBOL_GPL(nft_fib_dump);
nft_fib.c:EXPORT_SYMBOL_GPL(nft_fib_store_result);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_get_eval);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_set_eval);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_policy);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_get_init);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_set_validate);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_set_init);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_get_dump);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_set_dump);
nft_meta.c:EXPORT_SYMBOL_GPL(nft_meta_set_destroy);
nft_reject.c:EXPORT_SYMBOL_GPL(nft_reject_policy);
nft_reject.c:EXPORT_SYMBOL_GPL(nft_reject_validate);
nft_reject.c:EXPORT_SYMBOL_GPL(nft_reject_init);
nft_reject.c:EXPORT_SYMBOL_GPL(nft_reject_dump);
nft_reject.c:EXPORT_SYMBOL_GPL(nft_reject_icmp_code);
nft_reject.c:EXPORT_SYMBOL_GPL(nft_reject_icmpv6_code);
utils.c:EXPORT_SYMBOL(nf_ip_checksum);
utils.c:EXPORT_SYMBOL(nf_ip6_checksum);
utils.c:EXPORT_SYMBOL_GPL(nf_checksum);
utils.c:EXPORT_SYMBOL_GPL(nf_checksum_partial);
utils.c:EXPORT_SYMBOL_GPL(nf_route);
x_tables.c:EXPORT_SYMBOL(xt_register_target);
x_tables.c:EXPORT_SYMBOL(xt_unregister_target);
x_tables.c:EXPORT_SYMBOL(xt_register_targets);
x_tables.c:EXPORT_SYMBOL(xt_unregister_targets);
x_tables.c:EXPORT_SYMBOL(xt_register_match);
x_tables.c:EXPORT_SYMBOL(xt_unregister_match);
x_tables.c:EXPORT_SYMBOL(xt_register_matches);
x_tables.c:EXPORT_SYMBOL(xt_unregister_matches);
x_tables.c:EXPORT_SYMBOL(xt_find_match);
x_tables.c:EXPORT_SYMBOL_GPL(xt_request_find_match);
x_tables.c:EXPORT_SYMBOL_GPL(xt_request_find_target);
x_tables.c:EXPORT_SYMBOL_GPL(xt_data_to_user);
x_tables.c:EXPORT_SYMBOL_GPL(xt_match_to_user);
x_tables.c:EXPORT_SYMBOL_GPL(xt_target_to_user);
x_tables.c:EXPORT_SYMBOL_GPL(xt_find_revision);
x_tables.c:EXPORT_SYMBOL(xt_check_proc_name);
x_tables.c:EXPORT_SYMBOL_GPL(xt_check_match);
x_tables.c:EXPORT_SYMBOL(xt_check_table_hooks);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_add_offset);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_flush_offsets);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_calc_jump);
x_tables.c:EXPORT_SYMBOL(xt_compat_init_offsets);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_match_offset);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_match_from_user);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_match_to_user);
x_tables.c:EXPORT_SYMBOL(xt_compat_check_entry_offsets);
x_tables.c:EXPORT_SYMBOL(xt_check_entry_offsets);
x_tables.c:EXPORT_SYMBOL(xt_alloc_entry_offsets);
x_tables.c:EXPORT_SYMBOL(xt_find_jump_offset);
x_tables.c:EXPORT_SYMBOL_GPL(xt_check_target);
x_tables.c:EXPORT_SYMBOL_GPL(xt_copy_counters);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_target_offset);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_target_from_user);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_target_to_user);
x_tables.c:EXPORT_SYMBOL(xt_alloc_table_info);
x_tables.c:EXPORT_SYMBOL(xt_free_table_info);
x_tables.c:EXPORT_SYMBOL_GPL(xt_find_table_lock);
x_tables.c:EXPORT_SYMBOL_GPL(xt_request_find_table_lock);
x_tables.c:EXPORT_SYMBOL_GPL(xt_table_unlock);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_lock);
x_tables.c:EXPORT_SYMBOL_GPL(xt_compat_unlock);
x_tables.c:EXPORT_SYMBOL_GPL(xt_tee_enabled);
x_tables.c:EXPORT_SYMBOL(xt_counters_alloc);
x_tables.c:EXPORT_SYMBOL(xt_table_get_private_protected);
x_tables.c:EXPORT_SYMBOL_GPL(xt_replace_table);
x_tables.c:EXPORT_SYMBOL_GPL(xt_register_table);
x_tables.c:EXPORT_SYMBOL_GPL(xt_unregister_table);
x_tables.c:EXPORT_SYMBOL_GPL(xt_hook_ops_alloc);
x_tables.c:EXPORT_SYMBOL_GPL(xt_proto_init);
x_tables.c:EXPORT_SYMBOL_GPL(xt_proto_fini);
x_tables.c:EXPORT_SYMBOL_GPL(xt_percpu_counter_alloc);
x_tables.c:EXPORT_SYMBOL_GPL(xt_percpu_counter_free);
xt_RATEEST.c:EXPORT_SYMBOL_GPL(xt_rateest_lookup);
xt_RATEEST.c:EXPORT_SYMBOL_GPL(xt_rateest_put);










netfilter与iptables的实现原理。





























