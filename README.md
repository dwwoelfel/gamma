# gamma

Gamma simplifies an essential aspect of graphics programming: shaders. 

Graphics shaders are so complex and painful to program by hand that most significant graphics systems use some form of metaprogramming. However, these metaprogramming systems are often adhoc and limited to specific patterns. In the end, they don't solve the problem of GLSL's lack of fundamental programming affordances. 

Gamma lets you use a powerful tool to abstract and metaprogram shaders: Clojurescript. Gamma represents the GLSL AST as composable data, and lets you use virtually any Clojure programming pattern to abstract your shaders. 

Gamma is suitable for use by applications. However, its greater potential lies as a basis for graphics libraries. It makes it far easier to write these libraries, and also provides a basis for interoperability. 

Read the [rationale](https://github.com/kovasb/gamma/wiki/Gamma-Rationale).


# API

Gamma lets you do two things: A) construct a GLSL AST as clojure data, and B) compile it to a GLSL program string

The value of Gamma is that it lets you use Clojure to abstract the process in step A. You can use functions, procotols, multimethods, datastructures, etc to abstract the GLSL. Your Clojure program runs, and the GLSL AST is produced. Gamma does NOT try to tanspile Clojure to GLSL.


## Constructing GLSL 

##### GLSL represented as Clojure maps. Generate maps with constructor functions.  

Gamma represents the GLSL AST as Clojure maps.  The functions in gamma.api are convenience functions for constructing the maps:

```clojure
(require '[gamma.api :as g])
```

```clojure
(g/sin 1)
=>
{:tag :term, :head :sin, :id {:tag :id, :id 1}, :type :float,
  :body ({:tag :term, :head :literal, :value 1, :type :float, :id {:tag :id, :id 2}})}
```

Each GLSL operator, function, or type constructor has an equivalent function in gamma.api. 

##### GLSL Input/ouput variables

The different species of GLSL input/output variables also have constructors:

```clojure
;; attribute 
(g/attribute "a_Attr" :float)
;; uniform 
(g/uniform "u_Uniform" :mat4)
;; varying 
(g/varying "v_Varying" :float :highp)
```

To refer to a input variable with the AST, simply create it and pass it to an AST constuctor: 

```clojure 
(g/sin (g/attribute "a_Attr" :float))
```

##### GLSL If Statements are nestable expressions

In Gamma, we represent if-statements as expressions, so we can nest if's inside of other expressions:

```clojure 
(g/sin (g/if (g/attribute "b_Bool" :bool) 1 2))
```

##### Factor your AST with functions 

Gamma allows you to employ vanilla Clojure programming to build up the ASTs.

Insert arbitary helper functions to construct pieces of the tree:

```clojure
;; start with 
(g/+ 1 (g/+ 2 3))

;; create helper
(defn my-helper [x] (g/+ x 3))

;; refactor tree using helper:
(g/+ 1 (my-helper 2))
```

##### Use Clojure's binding forms 

To reuse an expression in multiple places, use let, or any other binding form:
```clojure
(let [x (g/length (g/attribute "a_Vec4" :vec4))]
  (g/vec3 x x x))
  
;; equivalent to 
(g/vec3 
    (g/length (g/attribute "a_Vec4" :vec4))
    (g/length (g/attribute "a_Vec4" :vec4))
    (g/length (g/attribute "a_Vec4" :vec4)))
```

Gamma's compiler will ensure that the (g/length ...) expression will only be evaluated once. This frees you from having to think about intermediary variables within the AST and their impact on performance. 

##### Higher-order AST construction

The Gamma AST is simple data and composes cleanly. Most things you can imagine doing will just work. 

```clojure
;; Some higher-order AST construction

(reduce g/+ 0 [1 2 3 4])

(apply g/vec4 (map #(g/clamp % 0 1) [0 0.5 1 2]))
```

##### Use datastructures to convey AST fragments

You can pass around AST fragments however you want, including inside datastructure:

```clojure

(def x {:color (g/vec3 0 0 1) :alpha 0.5})

(defn conj-alpha [c]
  (g/vec4 (:color c) (:alpha c)))
  
(conj-alpha x)  
```

If this looks like very basic programming, its because it is. However, these forms of abstraction are unavailable in raw GLSL, or in many of the languages and tools dedicated to wrapping it. 


### Differences from GLSL

#### No binding forms 

You don't create bindings or assignments within the AST.

If you want to pass an expression to multiple places, just do with Clojure:

```clojure
(defn my-abs [a b]
  (let [x (g/+ a b)]
    (g/if (g/< 0 x)
       (g/* -1 x) 
       x)))
```

#### Statements are expressions

Unlike ordinary GLSL, Gamma's AST is expression oriented. 

In particular, if-statements are expressions that can be nested in other expressions.

```clojure
(g/+ 1 (g/if conditional-expr 2 3))
```


This duplicates the (g/+ a b) AST fragment in multiple places, but Gamma's compiler will optimize that duplication away.

#### No for loops 

WebGL only supports for loops that are un-rollable. You can unroll these yourself trivially with higher-order functions.

For example, map a function across each element of a GLSL vector and produce a new GLSL vector:

```clojure
(defn map-over-vec4 [f v]
 (apply g/vec4 (for [i (range 4)] (f (g/part v i)))))
``` 

## Compiling to GLSL program strings






## License

Copyright © 2014 Kovas Boguta

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
