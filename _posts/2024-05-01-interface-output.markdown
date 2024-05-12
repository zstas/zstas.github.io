---
layout: post
title: "VPP interface output"
date:   2024-05-01 12:00:00 +0100
categories: jekyll update
---

## Subject
Developing a plugin for VPP always give new challenges. Lately I've been working on a code that directly sends traffic to interfaces and tunnels. This memo is a note about how properly forward packets to a desired interface.

# `interface-output` node
This is the most simple one. You have to prepare a proper packet (e.g. with ethernet header for an ethernet interface).
```
typedef enum {
  MY_NODE_NEXT_INTERFACE,
  MY_NODE_N_NEXT,
} my_node_next_t;


VLIB_NODE_FN (pppoe_cp_dispatch_node) (vlib_main_t * vm,
                    vlib_node_runtime_t * node,
                    vlib_frame_t * from_frame)
{
  u32 n_left_from, next_index, * from, * to_next;

  from = vlib_frame_vector_args (from_frame);
  n_left_from = from_frame->n_vectors;

  next_index = node->cached_next_index;

  while (n_left_from > 0)
  {
    u32 n_left_to_next;

    vlib_get_next_frame (vm, node, next_index,
			   to_next, n_left_to_next);

    while (n_left_from > 0 && n_left_to_next > 0)
    {
	u32 bi0;
	u32 next0;

	vlib_buffer_t * b0;

    bi0 = from[0];
	to_next[0] = bi0;
	from += 1;
	to_next += 1;
	n_left_from -= 1;
	n_left_to_next -= 1;

	b0 = vlib_get_buffer (vm, bi0);

    // code to select a proper interface

    next0 = MY_NODE_NEXT_INTERFACE;
    vnet_buffer(b0)->sw_if_index[VLIB_TX] = sw_if_index0;

    // omit trace

	vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
	                                 to_next, n_left_to_next,
                                     bi0, next0);
    }

    vlib_put_next_frame (vm, node, next_index, n_left_to_next);
  }

  return from_frame->n_vectors;
}


VLIB_REGISTER_NODE (my_node) = {
  .name = "my-node",
  //...
  .n_next_nodes = MY_NODE_N_NEXT,
  .next_nodes = {
    [0] = MY_NODE_NEXT_INTERFACE,
  },
};
```

# `ip4-rewrite` node
Use this node if you want to skip ip4 lookup but to use adjacency (e.g. `10.0.0.1 eth0`). You would need to specify this adjacency and then dispatch packet to this node. You can grab an adjacency by calling for instance `adj_nbr_add_or_lock`.

This node will check ttl and csum, add rewrite (e.g. ethernet header).

```
vnet_buffer (b[0])->ip.adj_index[VLIB_TX];
```

# `ip4-midchain` node
The same as previous but uses midchain adjacencies - for encapsulating packets and that use fixups. So, using this we can send packets to tunnel interfaces. Also, it will compute inner csum.

# `tunnel-output` and `adj-midchain-tx` nodes
Those 2 are basically the same and they work after `ip4-midchain` node and just adjust adjacency in a buffer to a next one, like after encapping a packet to a tunnel we'd need to add rewrite for an ethernet header. Therefore, this one is not what we want to use.

