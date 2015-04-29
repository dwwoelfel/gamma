# gamma

Gamma simplifies an essential aspect of graphics programming: shaders. It represents shaders as composable Clojurescript data, giving you Clojurescript's power to define and manipulate them. 

Graphics shaders are so complex and painful to program by hand that most significant graphics systems use some form of metaprogramming. Typically, they provide a more specific programming model, such as dataflow, that parameterizes the complexity of GLSL. This allows faster programming of a certain pattern or set of patterns, but falls short of providing the user with a substrate they can continue to abstract.

Gamma plays a different game. It simplifies GLSL itself, so that building abstractions on top becomes easy. Instead of imposing its own abstractions, Gamma enables using a powerful tool - Clojurescript - to build what you need. Furthermore, it provides the foundation for these user-land abstractions to interoperate, setting the stage for a la carte graphics libraries. Finally, it allows creating shaders in reponse to input, allowing the graphics pipeline to dictate its own variable names, data formats, and desired computations to GLSL, rather than than other way around. 

Gamma will be particularly useful to authors of graphics libraries, and graphics applications developers seeking greater flexibility and simplicity in their pipeline. 

Gamma is currently implemented for WebGL's GLSL and Clojurescript.

Read the [full rationale](https://github.com/kovasb/gamma/wiki/Gamma-Rationale).

Look at the tests. 

See examples. 

WebGL/GLSL references: [WebGL Cheatsheet](https://www.khronos.org/files/webgl/webgl-reference-card-1_0.pdf), [WebGL Spec](https://www.khronos.org/registry/webgl/specs/latest/1.0/), [OpenGL ES Shading Language 1.0 Spec](https://www.khronos.org/files/opengles_shading_language.pdf), [WebGL Programming Guide (Book)](http://www.amazon.com/WebGL-Programming-Guide-Interactive-Graphics/dp/0321902920)


# API

Gamma lets you do two main things:
- Construct a GLSL AST as clojure data
  - require gamma.api    
- Compile it to a GLSL program string 
  - require gamma.program 

There are also namespaces for AST pretty printing (gamma.print) and tooling (gamma.tools). 

## Constructing GLSL 

##### GLSL represented as Clojure maps. Generate maps with constructor functions.  

The GLSL AST is represented as Clojure maps with certain keys. 

Instead of entering the maps directly, use the constructor functions provided in gamma.api:

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

##### Compose constructor functions to buld the AST

Building the AST is just a matter of composing constructor functions:

```clojure
(g/clamp (g/sin s) 0.25 0.5)
```

It doesn't really matter how the AST comes together, just flow data to where it is needed.

```clojure
;; create some AST fragments and hang on to them
(def x {:partA (g/sin 1) :partB (g/cos a)})
;; get AST fragments and put them where we want
(g/clamp (:partA x) 0 (:partB x))
```

Feel free to use whatever abstractions you want for building up the tree. Just remember that GLSL is a typed language, and its functions and operations have type signatures that need to be respected. 

##### Types are checked and inferred by constructor functions

Constructor functions typecheck their arguments and infer their own types:

```clojure
(:type (sin 1.0))
=> :float
(:type (sin (vec3 0.0 0.0 1.0))
=> :vec3
```

Passing the wrong type results in an exception:

```clojure
(sin true)
=> 
```



##### GLSL Input/Ouput variables are also maps with constructor functions. 

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
