
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
134
135
136
137
138
139
140
141
142
143
144
145
146
147
148
149
150
151
152
153
154
155
156
157
158
159
160
161
162
163
164
165
166
167
168
169
170
171
172
173
174
175
176
177
178
179
180
181
182
183
184
185
# MemorySSA - Dead Store Elimination
## Directory description

## Compiling the demo pass 
```bash
clang++ -std=c++17 -fPIC -shared MemorySSADemo.cpp -o dylibs/libMemorySSADemo.dylib $(llvm-config --cxxflags --ldflags) -lLLVM 
```
Compiling the test examples to LLVM IR 
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/demo1.c -o irFiles/demo1.ll

clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/demo2.c -o irFiles/demo2.ll

clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/test1.c -o irFiles/test1.ll

clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/branch_example.c -o irFiles/branch_example.ll

clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/array_loop1.c -o irFiles/array_loop1.ll
```


## Running the pass with mem2reg 
```bash 
opt -load-pass-plugin dylibs/libMemorySSADemo.dylib \
-passes=mem2reg,memssa-demo \
irFiles/demo1.ll \
-disable-output
```
```bash 
Analyzing function: demo
BasicBlock: 
  1 = MemoryDef(liveOnEntry)
  2 = MemoryDef(1)
```
```bash
opt -load-pass-plugin dylibs/libMemorySSADemo.dylib \
    -passes=mem2reg,memssa-demo \
    irFiles/demo2.ll \
    -disable-output
```
```bash
Analyzing function: demo
BasicBlock: 
  1 = MemoryDef(liveOnEntry)
  2 = MemoryDef(1)
  MemoryUse(2)
```
```bash 
opt -load-pass-plugin dylibs/libMemorySSADemo.dylib \
    -passes=mem2reg,memssa-demo \
    irFiles/array_loop1.ll \
    -disable-output
```
```bash
Analyzing function: array_loop
BasicBlock: 
BasicBlock: 
  MemoryPhi for block :
    from : 0 = MemoryDef(liveOnEntry)
    from : 2 = MemoryDef(1)
BasicBlock: 
  MemoryUse(3)
  1 = MemoryDef(3)
  MemoryUse(1)
  2 = MemoryDef(1)
BasicBlock: 
BasicBlock: 
```
```bash
opt -load-pass-plugin dylibs/libMemorySSADemo.dylib \
    -passes=mem2reg,memssa-demo \
    irFiles/branch_example.ll \
    -disable-output
```
```bash
Analyzing function: branch_example
BasicBlock: 
  1 = MemoryDef(liveOnEntry)
BasicBlock: 
  2 = MemoryDef(1)
BasicBlock: 
  3 = MemoryDef(1)
BasicBlock: 
  MemoryPhi for block :
    from : 2 = MemoryDef(1)
    from : 3 = MemoryDef(1)
  MemoryUse(5)
  4 = MemoryDef(5)
```
```bash
opt -passes=mem2reg irFiles/demo1.ll -S -o irFiles/demo1_simplified.ll

opt -passes=mem2reg irFiles/demo2.ll -S -o irFiles/demo2_simplified.ll

opt -passes=mem2reg irFiles/test1.ll -S -o irFiles/test1_simplified.ll

opt -passes=mem2reg irFiles/branch_example.ll -S -o irFiles/branch_example_simplified.ll

opt -passes=mem2reg irFiles/array_loop1.ll -S -o irFiles/array_loop1_simplified.ll
```

## Pass extension for graphical view of the MemorySSA graph.
```bash
clang++ -std=c++17 -fPIC -shared MemorySSAGraph.cpp -o dylibs/libMemorySSAGdylib \
$(llvm-config --cxxflags --ldflags --system-libs --libs core analysis passes)
```
```bash
opt -load-pass-plugin dylibs/libMemorySSAGdylib \
    -passes=memssa-demo \
    irFiles/demo1_simplified.ll \
    -disable-output
```
```bash 
opt -load-pass-plugin dylibs/libMemorySSADSE.dylib \
    -passes=mem2reg,memssa-dse \
    tests/task5_tests/mayAnalysis/irFiles/test1.ll \
    -disable-output
```
```bash
dot -Tpdf mssa_demo1.dot -o graphs/mssa_demo1.pdf
```
```bash
opt -load-pass-plugin dylibs/libMemorySSAGdylib \
    -passes=memssa-demo \
    irFiles/demo2_simplified.ll \
    -disable-output
```
```bash
dot -Tpdf mssa_demo.dot -o graphs/mssa_demo2.pdf
```
```bash
opt -load-pass-plugin dylibs/libMemorySSAGdylib \
    -passes=memssa-demo \
    irFiles/array_loop1_simplified.ll \
    -disable-output
```
```bash
dot -Tpdf mssa_array_loop.dot -o graphs/mssa_array_loop.pdf
```
```bash
opt -load-pass-plugin dylibs/libMemorySSAGdylib \
    -passes=memssa-demo \
    irFiles/branch_example_simplified.ll \
    -disable-output
```
```bash
dot -Tpdf mssa_branch_example.dot -o graphs/mssa_branch_example.pdf
```

## Intraprocedural dead store elimination (DSE) pass 

Implemented the lecture DSE algorithm using MemorySSA; restricted MustAlias to Ptr == PrevPtr and only considered same-basic-block stores, 
which makes post-dominance trivial. Intervening uses are checked by scanning between the two stores in that block for loads from the same pointer.

``bash
clang++ -std=c++17 -fPIC -shared DeadStoreElimination/DeadStoreE.cpp -o dylibs/libMemorySSADSE.dylib \
$(llvm-config --cxxflags --ldflags --system-libs --libs core analysis passes)
```

##Compiling the test cases which is in the `tests/mayAnalysis` and `tests/mustAnalysis` directories. The IR files are kept in `irFiles` folder in each of the 
may and must analysis folder in the `tests` directory

###May Analysis
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/task5_tests/mayAnalysis/test1.c -o tests/task5_tests/mayAnalysis/irFiles/test1.ll 
```
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/task5_tests/mayAnalysis/test2.c -o tests/task5_tests/mayAnalysis/irFiles/test2.ll
```
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/task5_tests/mayAnalysis/test3.c -o tests/task5_tests/mayAnalysis/irFiles/test3.ll
```

### Must Analysis
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/task5_tests/mustAnalysis/test1.c -o tests/task5_tests/mustAnalysis/irFiles/test1.ll 
```
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/task5_tests/mustAnalysis/test2.c -o tests/task5_tests/mustAnalysis/irFiles/test2.ll
```
```bash
clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm tests/task5_tests/mustAnalysis/test3.c -o tests/task5_tests/mustAnalysis/irFiles/test3.ll
```

## Run the pass on the IR files 
