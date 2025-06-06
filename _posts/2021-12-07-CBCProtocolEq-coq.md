---
layout: post
title: CBC Casper Protocols Formal Definition in Coq
subtitle: Computer Science
tags: [computer science, blockchain, consensus protocol, CBC Casper, formalization, Coq]
comments: true
---

This is part of a joint work from the Formal Verification in Blockchain reading group, which consists of [Barnabé Monnot](https://barnabemonnot.com/), [Zhangsheng Lai](https://zunction.github.io/) and myself.    

In this post, we will be understanding and covering the important definitions and properties from the typeclass `CBCProtocolEq` from the [Coq](https://coq.inria.fr/) source code, `Protocol.v`, presented in the paper, [Formalizing Correct-by-Construction Casper in Coq](https://www.researchgate.net/publication/343704844_Formalizing_Correct-by-Construction_Casper_in_Coq), by authors from [Runtime Verification Inc](https://runtimeverification.com/).
The full source code can be found in [here](https://runtimeverification.github.io/casper-cbc-proof-docs/docs/latest/alectryon/toc.html). 
We will also be referencing from the paper, [Introducing the "Minimal CBC Casper" Family of Consensus Protocols](https://github.com/cbc-casper/cbc-casper-paper/blob/master/cbc-casper-paper-draft.pdf), by authors from [Ethereum Research](https://ethresear.ch/) as the code covers the formalized version of the definitions and properties in the paper as well.
We will be mapping from the formalized definitions and properties found in `Protocol.v` to the corresponding definitions and properties in either [Formalizing Correct-by-Construction Casper in Coq](https://www.researchgate.net/publication/343704844_Formalizing_Correct-by-Construction_Casper_in_Coq) or [Introducing the "Minimal CBC Casper" Family of Consensus Protocols](https://github.com/cbc-casper/cbc-casper-paper/blob/master/cbc-casper-paper-draft.pdf).
We will provide necessary details for each of the mappings to provide a better understanding for the readers but will not be going in depth for most of the proofs presented.
Interested readers can either run through the Alectryon proof movies in [here](https://runtimeverification.github.io/casper-cbc-proof-docs/docs/latest/alectryon/toc.html). 
For readers who would wish to understand more about CBC Casper specifications, you can read the [post](https://barnabemonnot.com/posts/2018/11/14/casper-cbc.html) by Barnabé, where he explains the protocol specifications in details with illustrative diagrams.
Moreover, readers, who want to understand the formal proofs of the 5 main theorems regarding the safety properties from [Introducing the "Minimal CBC Casper" Family of Consensus Protocols](https://github.com/cbc-casper/cbc-casper-paper/blob/master/cbc-casper-paper-draft.pdf), can read the [post](https://zunction.github.io/blog/2021/safety-proofs/) by Zhangsheng Lai. 


Recommended order of reading our posts:    
Part 1. [Safety of CBC Casper consensus](https://hackingresear.ch/cbc-safety/) by Barnabé Monnot    
Part 2. [CBC Casper Protocols Formal Definition in Coq](https://jinxinglim.github.io/2021-12-07-CBCProtocolEq-coq/) by Jin Xing Lim    
Part 3. [Safety Proofs for the minimal CBC Casper family](https://zunction.github.io/blog/2021/safety-proofs/) by Zhangsheng Lai

---

### Correct-by-Construction (CBC) Casper Protocols Formal Definition

**Consensus protocols** can be thought of as "rules" used by nodes in a distributed network to make consistent decisions, i.e., come to consensus, from possibly inconsistent alternative decisions from different nodes. For example, in a binary consensus protocol, where the decisions are 0 or 1, the decision 1 is said to be inconsistent with 0 and consistent with 1. In a blockchain consensus protocol, a decision on one block is considered as inconsistent with another when they are not in the same blockchain, and consistent when they are in the same chain. Typically, consensus protocols are to ensure several properties to hold within a blockchain, e.g., safety, liveness and non-triviality. In this blog post, we will just be looking at how CBC Casper protocols ensure **safety**, i.e., it is not possible for nodes to make inconsistent decisions/consensus values (more formal definition will be given later).

**Correct-by-Construction (CBC) Casper** is a partial specification for a family of consensus protocols and is formally defined as a type class, using the syntax `Class`, in the Coq proof assistant as shown below:

```coq
Class CBCProtocolEq :=
   {
      (** Consensus values equipped with reflexive transitive comparison **)
      consensus_values : Type;
      about_consensus_values : StrictlyComparable consensus_values;
      (** Validators equipped with reflexive transitive comparison **)
      validators : Type;
      about_validators : StrictlyComparable validators;
      (** Weights are positive reals **)
      weight : validators -> {r | (r > 0)%R};
      (** Threshold is a non-negative real **)
      t : {r | (r >= 0)%R};
      suff_val : exists vs, NoDup vs /\ ((fold_right (fun v r => (proj1_sig (weight v) + r)%R) 0%R) vs > (proj1_sig t))%R;
      (** States with equality and union **)
      state : Type;
      about_state : StrictlyComparable state;
      state0 : state;
      state_eq : state -> state -> Prop;
      state_union : state -> state -> state;
      state_union_comm : forall s1 s2, state_eq (state_union s1 s2) (state_union s2 s1);
      (** Reachability relation **)
      reach : state -> state -> Prop;
      reach_refl : forall s, reach s s;
      reach_trans : forall s1 s2 s3, reach s1 s2 -> reach s2 s3 -> reach s1 s3;
      reach_union : forall s1 s2, reach s1 (state_union s1 s2);
      reach_morphism : forall s1 s2 s3, reach s1 s2 -> state_eq s2 s3 -> reach s1 s3;
      (** Total estimator **)
      E : state -> consensus_values -> Prop;
      estimator_total : forall s, exists c, E s c;
      (** Protocol state definition as predicate **)
      prot_state : state -> Prop;
      about_state0 : prot_state state0;
      (** Equivocation weights from states **)
      equivocation_weight : state -> R;
      equivocation_weight_compat : forall s1 s2, (equivocation_weight s1 <= equivocation_weight (state_union s2 s1))%R;
      about_prot_state : forall s1 s2, prot_state s1 -> prot_state s2 ->
                                  (equivocation_weight (state_union s1 s2) <= proj1_sig t)%R -> prot_state (state_union s1 s2);
   }.
```

> **Remark 1**: Type classes in Coq works similarly to the record types in most programming languages. One of the main differences is that in most programming languages' record types, they are used to store multiple *data types*, but for Coq's type classes, they can store *properties* and *functions* as well. This is because Coq is built through dependent type theory and is a functional programming language where *propositions* are types as well.

> **Remark 2**: As mentioned above, propositions are types in Coq. In particular, they form a special collection of types, called `Prop`, which is a subtype (loosely speaking, subset) of the most general type, `Type`, which consists of other collections of types, such as `Set` (more can be found on the [Coq Typing Rules](https://coq.inria.fr/refman/language/cic.html)). 

We will slowly break down the type class, `CBCProtocolEq`, above and explain what data and properties are required for CBC Casper protocols (through the comments in each of the code snippets). CBC Casper is instantiated with 5 framework parameters to define concrete protocols:

1. **Validators** or **participating nodes**, `validators` (Definition 2.1 from [1]): A non-empty set, $\mathcal{V}$
    ```coq
   validators : Type;
   (** the idea of validators is abstracted to an arbitrary type, instead of defining it to a fixed set of names **)
   about_validators : StrictlyComparable validators;
   (** about_validators is the proof term of a proposition which tells us that you can compare validators and this comparison is reflexive and transitive **)
   ```
   
2. **Validator weights**, `weight` (Definition 2.2 from [1]): A function, $\mathcal{W}: \mathcal{V} \rightarrow \mathbb{R}\_+$, that assigns a weight to each validator
   ```coq
   weight : validators -> {r | (r > 0)%R};
   (** the function is defined as a sigma/pair type where for any input validator, the output will be a pair of types, **)
   (** where the first type is real number r and the second type is the property that r > 0 **)
   ```
   > **Remark**: One can think of this weight function, $\mathcal{W}$, as a function that maps from a validator to his/her amount of Ether he/she holds
   
3. **Byzantine-fault-tolerance threshold**, `t` (Definition 2.3 from [1]): A non-negative real number, $t$, strictly smaller than the total validator weights
   ```coq
   t : {r | (r >= 0)%R};
   (** t is defined as a sigma/pair type where the first type is real number r and the second type is the property that r >= 0 **)
   suff_val : exists vs, NoDup vs /\ ((fold_right (fun v r => (proj1_sig (weight v) + r)%R) 0%R) vs > (proj1_sig t))%R;
   (** suff_val is the proof term of a proposition that tells us that t must be strictly smaller than the total validator weights **)
   (** i.e., there exists a list of validators, vs, where there is no duplicate in vs and if you sum all weights of the validators in vs, it will be more than t **)
   ```
   > **Remark 1**: The CBC Casper protocol states are parametric in a Byzantine-fault-tolerance threshold, $t$.
   
   > **Remark 2**: The protocols will be guaranteed to have consensus safety tolerating up to $t$ Byzantine faults", measured by weight.
  
4. **Consensus values**, `consensus_values` (Definition 2.4 from [1]): A multi-element set, $\mathcal{C}$
   ```coq
   consensus_values : Type;
   (** the idea of consensus_values is abstracted to an arbitrary type, instead of defining it to a fixed set of blockchains **)
   about_consensus_values : StrictlyComparable consensus_values;
   (** about_consensus_values is the proof term of a proposition which tells us that you can compare consensus values and this comparison is reflexive and transitive **)
   ```
   > **Remark**: E.g., in binary consensus protocol, $\mathcal{C}=\\{0,1\\}$, while in a blockchain consensus protocol, $\mathcal{C}$ is the set of all possible blockchains.
   
5. **Estimator**, `E` (Definition 2.5 from [1]) A function $\varepsilon: \Sigma \rightarrow \mathcal(P)(\mathcal{C}) \setminus \emptyset$, which relates CBC Casper protocol states, $\Sigma$, (we will define this later) to the sets of consensus values, in particular, powerset of $\mathcal{C}, \mathcal{P}(\mathcal{C})$ excluding the empty set, $\emptyset$
   ```coq
   E : state -> consensus_values -> Prop;
   (** the function is defined as a proposition that takes in a state and a consensus value in this particular order **)
   estimator_total : forall s, exists c, E s c;
   (** estimator_total is a proof term of a proposition which tells us that for all state s, there exists a consensus value c, such that E s c hold **)
   ```
   > **Remark**: The Coq code presented above uses 3 similar terms, `state`, `prot_state` and `pstate`, which can be confusing for first-time readers. In summary, `state` represents protocol state as seen in [1], `prot_state` represents the property that a state is indeed a protocol state with equivocation fault tolerance `t` and `pstate` represents an actual protocol state with equivocation fault tolerance `t` (more on this later).

There are two things to take note of:

1. As seen from the code above, for each of the 5 parameters, other than defining its definition, we need to include the properties that come with it within the typeclass, `CBCProtocolEq`, as well, e.g., `about_validators` for `validators`, `about_consensus_values` for `consensus_values`, etc. Some of them, such as `suff_val` for `t`, are constraints of the corresponding definitions. However, when we abstract the ideas of `validators` and `consensus_values` to the typeclass `Type`, we need to provide them with properties (`StrictlyComparable` in these two cases) explicitly, which are required in the formal proofs later on. 
2. The functions, such as `weight`, `t` and `E`, are defined as sigma/pair type or proposition type instead of the regular function type, where we can use `Definition` and/or `Fixpoint` to define in Coq. The purpose of doing so is better facilitate the proving processes of the formal proofs which we will see later on. Defining them as functions does not guarantee the correctness of their specifications. On the other hand, when defining them as propositions guarantees their correctness (thats why properties mentioned in point 1 above must be defined) and can be used easily in the long run when proving the safety properties of the CBC Casper protocols.

These 5 framework parameters in turn define protocol states and messages, which are defined in Section 2.2 of [1], in particularly as Definition 2.6 and 2.7. We will not be going in depth to these definitions because the code abstracts and removes the protocol-specific features of messages and validator weights, and uses the idea of equivocation to define what a protocol state is (this is where the difference between `state` and `prot_state` comes in). This is sufficient to show the safety properties of the CBC Casper protocols, which we will discuss later. First, let us understand what equivocation means.

- **Equivocation**: Equivocating nodes/validators (by definition) exclusively send different (and hence, contradicting) nodes messages from independent protocol executions, e.g., a node sending the two different messages that say both chain A and chain B are the "right" chain to follow. As a result, a large number of equivocation faults (per state) is fundamentally enough to cause consensus failure in any consensus protocol.

CBC Casper protocols are Byzantine fault-tolerant, `t`, with respect to equivocation. They can be instantiated to define concrete protocols, such as [Casper the Friendly Finality Gadget](https://arxiv.org/abs/1710.09437), that share the same proofs of desired protocol properties: safety and non-triviality. 
1. **Safety states** that with not too many equivocating nodes, all participating nodes decide on the same consensus value. Namely, nodes “decide” on the same consensus value given not too many Byzantine nodes. 
2. **Non-triviality states** that it is always possible for participating nodes to make inconsistent decisions on consensus values.

As mentioned previously, we will be focusing on the formal proofs of the safety properties of the states in CBC Casper protocols and will leave the non-triviality properties for future discussions. So now let us look at the remaining data and their properties within the typeclass `CBCProtocolEq` (through the comments in the code snippet below):

- **States**, `state`: Refers to the term "protocol state" in [1].
   ```coq
   state : Type;
   (** the idea of state is abstracted to an arbitrary type, instead of defining it in terms of messages as shown in Definition 2.6 and 2.7 **)
   about_state : StrictlyComparable state;
   (** about_state is the proof term of a proposition which tells us that you can compare state and this comparison is reflexive and transitive **)
   state0 : state;
   (** state0 is the initial state of the type, "state" **)
   state_eq : state -> state -> Prop;
   (** state_eq defines state equality that takes in 2 (input) states and output an equality proposition **)
   state_union : state -> state -> state;
   (** state_union defines the operation that you can union 2 states **)
   (** i.e., if s1 and s2 are states, then (state_union s1 s2) is a state **)
   state_union_comm : forall s1 s2, state_eq (state_union s1 s2) (state_union s2 s1);
   (** state_union is communtative **)
   ```
   
- **Reachability Relation**, `reach` (Combines Definition 2.7 and 2.8 of [1]):
   ```coq
   reach : state -> state -> Prop;
   (** reach defines reachability relation that takes in 2 (input) states and output a reachability proposition **)
   reach_refl : forall s, reach s s;
   (** reach is reflexive **)
   reach_trans : forall s1 s2 s3, reach s1 s2 -> reach s2 s3 -> reach s1 s3;
   (** reach is transitive **)
   reach_union : forall s1 s2, reach s1 (state_union s1 s2);
   (** reach_union tells us that for any states s1 s2, s1 can reach/transit to the (state_union s1 s2) **)
   reach_morphism : forall s1 s2 s3, reach s1 s2 -> state_eq s2 s3 -> reach s1 s3;
   (** reach_morphism tells us that for any states s1 s2 s3 are states, if s1 can reach s2 and s2 and s3 are equal in terms of state_eq, then s1 can reach s3 **)
   ```
   > **Remark**: The `reach` and its properties in `CBCProtocolEq` provide us a constructive way of protocol state transition (Definition 2.8 of [1]), which is to recursively built through `state_union`, shown in Definition 2.7 of [1].  
   
- **Equivocation Weight**, `equivocating_weight` (Definition 2.10 of [1]):    
   $F: \Sigma \rightarrow \mathbb{R}\_+$    
   $F(\sigma) := \sum_{v \in E(v) } \mathcal{W}(v)$ where $E(v) :=$ the set of equivocating validators in state $\sigma$     
   F is monotonic function, i.e., $\sigma\_1 \subseteq \sigma\_2 \implies F(\sigma\_1) \leq F(\sigma\_2)$.
   ```coq
   equivocation_weight : state -> R;
   (** equivocation_weight takes in an input, state, and output a real number, R **)
   equivocation_weight_compat : forall s1 s2, (equivocation_weight s1 <= equivocation_weight (state_union s2 s1))%R; 
   (** equivocation_weight_compat defines the monotonicity of F through state_union **)
   ```

- **Property of `state` Having Equivocation Fault Tolerence `t`**, `prot_state` (Part of Definition 2.12 of [1]): Refers to the property of being a protocol state with equivocation fault tolerence `t`, i.e., the property of satisfying $F(\sigma) \leq t$
   ```coq
   prot_state : state -> Prop;
   (** prot_state is a property/proposition that tells us that the input state is a prot_state **)
   about_state0 : prot_state state0;
   (** about_state0 tells us that the initial state0 have the property prot_state **)
   about_prot_state : forall s1 s2, prot_state s1 -> prot_state s2 -> 
                        (equivocation_weight (state_union s1 s2) <= proj1_sig t)%R -> prot_state (state_union s1 s2);
   (** about_prot_state tells us that for any states s1 s2, if s1 and s2 have the property prot_state, and equivocation_weight of (state_union s1 s2) is less than or equal to (some predefined) t, then (state_union s1 s2) has the property prot_state **)
   ```

With the necessary formal definitions and properties defined for the typeclass `CBCProtocolEq`, we are ready to provide a property of reachability, `reach_total`, definition of an actual protocol state with equivocation fault tolerance `t`, `pstate` and its properties and that this `pstate` type is a [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set). Some of these properties may seem trivial to us, but they are required to formally defined in Coq to be used in the safety proofs (later on). 

- **Theroem `reach_total`**: (within the scope of `CBCProtocolEq`) For all states `s`, there exists state `s'` such that `s` can reach `s'`.    
  *Proof idea:* Constructive proof by providing the fact that `s` can reach `state_union s s`.
  ```coq
  Theorem reach_total `{CBCProtocolEq} :
  forall s, exists s', reach s s'.
  Proof. intro s. exists (state_union s s). apply (reach_union s s). Qed.
  ```
  
- **Protocol state with Equivocation Fault Tolerence `t`**, `pstate` (Definition 2.12 of [1]):
   ```coq
   Definition pstate `{CBCProtocolEq} : Type := {s : state | prot_state s}.
   (** pstate is defined as a sigma/pair type where the first type is state s and the second type is the proposition that s has prot_state property **)
   Definition pstate_proj1 `{CBCProtocolEq} (p : pstate) : state := proj1_sig p.
   (** pstate_proj1 projects the first type of pstate, state s, which satisfies prot_state s **)
   Coercion pstate_proj1 : pstate >-> state.
   (** Coercion of pstate_proj1 allows the first type of pstate, state s, to inherit the properties of state defined previously **)
   ```
   
- **`pstate` is `PartialOrder`**
  - Formal Definition of `PartialOrder`:
    ```coq
    Class PartialOrder (A : Type) :=
    (** for type A to be PartialOrder, the following properties must hold: **)
    { A_eq_dec : forall (a1 a2 : A), {a1 = a2} + {a1 <> a2};
      (** existence of decidable equality for A **)
      A_inhabited : exists (a0 : A), True; (* ? *)
      (** A must be inhabited, i.e., non-empty **)
      A_rel : A -> A -> Prop;
      (** existence of A_rel that can compare 2 terms of type A **)
      A_rel_refl :> Reflexive A_rel;
      (** A_rel must be reflexive **)
      A_rel_trans :> Transitive A_rel;
      (** A_rel must be transitive **)
    }.
    ```
    
  - Proof that `pstate` is a `PartialOrder`:
    ```coq
    Lemma pstate_eq_dec `{CBCProtocolEq} : forall (p1 p2 : pstate), {p1 = p2} + {p1 <> p2}.
    Proof.
    intros p1 p2.
    assert (H_useful := about_state).
    now apply sigify_eq_dec.
    Qed.
    (** Proof that there is decidable equality for pstate **)
    
    Lemma pstate_inhabited `{CBCProtocolEq} : exists (p1 : pstate), True.
    Proof. now exists (exist prot_state state0 about_state0). Qed.
    (** Proof that the type pstate is inhabited, i.e., non-empty **)
    
    Definition pstate_rel `{CBCProtocolEq} : pstate -> pstate -> Prop :=
       fun p1 p2 => reach (pstate_proj1 p1) (pstate_proj1 p2).
    (** pstate_rel defines the relation taht compares 2 pstate **)
    
    Lemma pstate_rel_refl `{CBCProtocolEq} : Reflexive pstate_rel.
    Proof. red; intro p; now apply reach_refl. Qed.
    (** Proof that pstate_rel is reflexive **)
    
    Lemma pstate_rel_trans `{CBCProtocolEq} : Transitive pstate_rel.
    Proof.
    red; intros p1 p2 p3 H_12 H_23.
    destruct p1 as [p1 about_p1];
    destruct p2 as [p2 about_p2];
    destruct p3 as [p3 about_p3];
    simpl in *.
    now apply reach_trans with p2.
    Qed.
    (** Proof that pstate_rel is transitive **)
    
    Instance level0 `{CBCProtocolEq} : PartialOrder pstate :=
    { A_eq_dec := pstate_eq_dec;
      A_inhabited := pstate_inhabited;
      A_rel := pstate_rel;
      A_rel_refl := pstate_rel_refl;
      A_rel_trans := pstate_rel_trans;
    }.
    (** Proof that pstate is a PartialOrder **)
    ```

---

This brings us to the end of this post. As we are just focusing on the safety proofs from [1], it suffices for us to understand the typeclass `CBCProtocolEq` from `Protocol.v`. However, for the formal proofs of weak and strong non-trivality, one needs to understand the other typeclasses, such as `FullNode`, `LightNode` and `PartialOrderNonLCish`, presented in the source code [4] and as explained in [2] itself.

---

### References

[1]. [Introducing the "Minimal CBC Casper" Family of Consensus Protocols](https://github.com/cbc-casper/cbc-casper-paper/blob/master/cbc-casper-paper-draft.pdf)

[2]. [Formalizing Correct-by-Construction Casper in Coq](https://www.researchgate.net/publication/343704844_Formalizing_Correct-by-Construction_Casper_in_Coq)

[3]. [Casper the Friendly Finality Gadget](https://arxiv.org/abs/1710.09437)

[4]. [Source code of the paper, Formalizing Correct-by-Construction Casper in Coq](https://runtimeverification.github.io/casper-cbc-proof-docs/docs/latest/alectryon/toc.html)

[5]. [Safety Proofs for the minimal CBC Casper family](https://zunction.github.io/blog/2021/safety-proofs/) post by Zhangsheng Lai

[6]. [Safety of CBC Casper consensus](https://hackingresear.ch/cbc-safety/) post by Barnabé Monnot
