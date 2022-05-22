# thinner sway #

This fork contains two small changes to sway's tiling behavior:

* prevent singletons from nesting
* (in the "autotiling" branch only) split windows on the long side by default (for bspwm-like tiling)


## Flattening details ##

Ideal flattening behavior is hard to pin down, but I've summarized my opinion with this "soft" rule:

**Soft rule for flattening:** _Containers should either be visible, or the split direction for a visible container._

A container is "visible" if it has multiple children. You can look at a group of windows in a horizontal layout and see that they are in a horizontal container. You can also visibly _select_ that container (it looks like selecting all of its children), so we should be able to split it in an arbitrary direction. So at the very least, we need to allow 1 layer of singletons (stuff like H[V[a b c]]).

A singleton containing a singleton (e.g. `H[H[app]]`) can't be seen. You can only know they exist if you remember creating them (or by experimenting). They're not easy to create, since i3/sway avoid creating them on splits; you have to "back into them" by removing containers, like `H[x V[y]] --> H[V[y]]`. So, they make navigation confusing.

The outer container doesn't serve any real purpose. It's not telling `app` how to split, it's only telling `app`'s invisible container how to split. So its only possible function is if you want to split `app` vertically but maintain its horizontal direction for future splits:

    focus parent && splitv && insert app2 : H[H[app*]] --> V[H[app] app2*]
However, the outer container does nothing before this command is run. The result is the exact same layout if we started from `H[app*]`, as H will be put into a new vertical split. 

It is conceptually and computationally simpler to regard singleton containers as "pending splits," which should exist only at the leaf level, rather than throughout the tree.  We should create them only as-needed (already done), modify their layouts instead of nesting (already done), and destroy them when possible (done in this fork and *sometimes* done in mainline i3/sway, see the next section). Therefore, the "hard rule" for flattening which this fork enforces is:

**Hard rule for flattening:** _If A and B are singleton containers and A > B > C, squash to A > C._

It does so on an as-needed basis whenever a container is moved or removed, by rearranging containers before a singleton is created. When removing a container with a unique sibling:
  1) If the sibling has an only child, presquash the sibling.
  
    A[con, B[nephew]] --(presquash)--> B[nephew, con] --(remove con)--> B[nephew]
  2) If the sibling will become an only child, presquash the parent.
  
    A[B[con, sibling]] --(presquash)--> B[con, sibling] --(remove con)--> B[sibling]

## Flattening in mainline i3/sway ## 
In mainline i3/sway, singleton containers are pruned in two ways:

* when splitting a singleton container, instead of adding a new container, just change the parent's layout 
    
      H[x] -> V[x] instead of H[V[x]]
* when moving containers, recursively flatten the entire workspace, squashing _alternating_ chains `A > B > A` where `B` is a singleton:
           
      H[... V[H[app]] ...] --> H[... app ...], H[a b V[H[c]]] --> H[a b c]

This ad hoc approach is a bit mysterious. Since it only cares if the middle container is a singleton, it has unexpected results (see Example 1). It's also only done to the target workspace when *moving* a split, but not the origin workspace or when *killing/removing* a split. It's also done recursively to the entire workspace, instead of only to the small neighborhood of containers which are affected by a single move. Since it doesn't flatten singleton chains unless they're alternating, you can easily inflate your tree by accident (see example 2).



## Example 1 ##
### starting layout: H[V[a* H[b c]] d] ###
![layout 1: V[a H[V[b* H[c d]] e]]](https://github.com/nseguin42/sway/blob/master/demos/layout_A_start.png)

### after 'move down' in mainline i3/sway: H[b a* c d] ### 
![layout 1 after 'move down' in mainline](https://github.com/nseguin42/sway/blob/master/demos/layout_A_post_mainline.png)
Notice that `d` has been resized, even though we only moved `a` within a branch which does not include `d`!

### after 'move down' in this fork: H[H[b a* c] d]
![layout 1 after 'move down' in this fork](https://github.com/nseguin42/sway/blob/master/demos/layout_A_post_fork.png)


## Example 2 ##
The unsquashed containers in mainline i3/sway arise from doing `kill x : A[x B[y]] --> A[B[y]]`. Here's a simple example of how this can lead to artifically inflated trees, especially when using a script that automatically alternates split directions.
### an alternating layout: H[V[H[V[H[V[H[V[H[a V[a H[a V[a H[a V[a H[a a]]]]]]]]]]]]]]] ###
![layout 2](https://github.com/nseguin42/sway/blob/master/demos/layout_B_start.png)
### after killing views from the bottom up (mainline): H[V[H[V[H[V[H[V[H[V[H[V[H[V[H[a]]]]]]]]]]]]]]] ###
![layout 2 after killing in a specific order in mainline](https://github.com/nseguin42/sway/blob/master/demos/layout_B_post_mainline.png)
### after killing views from the bottom up (in this fork): H[a] ###
![layout 2 after killing in a specific order in fork](https://github.com/nseguin42/sway/blob/master/demos/layout_B_post_fork.png)

## More examples of internal/invisible flattening ##
  
* H[H1[V[x y z]]] -> H[V[x y z]] 

      Singletons: H > H1, H1 > V
      Chains: H > H1 > V --> H > V
  
* H[V1[H1[V2[H2[x y]] z]]] -> H[V2[H2[x y]] z]

      Singletons: H > V1, V1 > H1, H1 > V2, V2 > H2
      Chains: H > V1 > H1 > V2 --> H > V2

* V1[a H1[V2[H2[b c]]] d] -> V1[a H1[H2[b c]] d]
 
      Singletons: H1 > V2, V2 > H2
      Chains: H1 > V2 > H2 --> H1 > H2

