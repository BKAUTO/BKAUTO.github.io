---
layout: post
title: "LeetCode 1192 Critical Connections in a Network"
subtitle: "查找集群内的「关键连接」"
date: 2020-10-07
author: "BKAUTO"
header-img: "img/post-bg-2020.jpg"
tags:
    - 算法与数据结构
    - Graph
    - Tarjan
---

### 题目描述
[1192. Critical Connections in a Network](https://leetcode-cn.com/problems/critical-connections-in-a-network/)

### 简单分析
这道题目的中文题解，大多上来就和你说[Tarjan算法](http://keyblog.cn/article-72.html)。
Tarjan算法是用来计算强联通分量的，这道题是一个无向图，虽然最终的实现算法与Tarjan高度相似，但在构思上最好还是有特化的思路。

#### 什么样的连接是「关键连接」？
对于这样一个无向图，当且仅当**边不在环上**时，边是「关键连接」。  

#### 边在环上
继续思考，如何判断「边不在环上」这件事？  
采用的方法和Tarjan很像：  
- 时间戳`visitTime`——记录访问顺序
- 最早可访问位置`backTrack`——节点直接或间接可访问的最早节点时间戳（回溯）  

> 不考虑节点和其父节点的连接，节点的backTrack回溯若大于其父节点的visitTime时间戳，这两节点构成的边不在环上。  

对于一个节点node和其父节点parent，`backTrack(node) > visitTime(parent)`代表node不能通过一个环找到先于parent访问过的其他节点，故node和parent构成的边不在环上。  

### C++代码实现

```c++
class Solution {
public:
    vector<vector<int>> criticalConnections(int n, vector<vector<int>>& connections) {
        vector<vector<int>> graph(n);
        vector<int> visitTime(n ,-1); //节点时间戳，记录节点访问顺序
        vector<int> backTrack(n ,-1); //最早可访问位置，节点直接或间接可访问的最早节点时间戳
        int time = 0;
        vector<vector<int>> res;
        for (auto edge : connections) { //根据连接关系构建邻接表
            graph[edge[0]].emplace_back(edge[1]);
            graph[edge[1]].emplace_back(edge[0]);
        }
        dfs(0, time, -1, visitTime, backTrack, graph, res);
        return res;
    }

    void dfs(int node, int& time, int parent, vector<int>& visitTime, vector<int>& backTrack, vector<vector<int>>& graph, vector<vector<int>>& res) {
        visitTime[node] = backTrack[node] = ++time; //初始化当前节点时间戳及最早可访问位置
        auto neighbors = graph[node];
        for (auto neighbor : neighbors) {
            if (neighbor == parent) continue;
            if (visitTime[neighbor] == -1) { //相邻节点未访问过
                dfs(neighbor, time, node, visitTime, backTrack, graph, res);
                backTrack[node] = min(backTrack[node], backTrack[neighbor]);
                if(backTrack[neighbor] > visitTime[node]) { //不在环上
                    res.emplace_back(vector<int>{node, neighbor});
                }
            }
            else {
                backTrack[node] = min(backTrack[node], backTrack[neighbor]);
            }
        }
    }
};
```

### 实现细节
#### DFS在计算什么？
类似Tarjan，深度优先搜索计算的是节点的时间戳`visitTime`和最早可访问位置`backTrack`。
    
所以，对于未曾访问过的节点，需要对其递归地执行DFS，一旦其计算出来的backTrack符合我们上一小节分析出的「边不在环上」条件，就是找到了「关键连接」。  
   
这一部分是算法的核心，对应代码21～28行。  

#### 访问过的相邻节点
代码30行，我们处理了访问过的相邻节点。  
    
这是因为backTrack随着我们对越来越多节点的访问，是在动态更新的。

#### 处理相邻节点时，为什么不考虑父节点？
21行，若相邻节点是父节点，continue。  
   
想一想，若与父节点的连接也考虑在内，最早可访问位置至少也是父节点的时间戳，25行的判断就无意义了。

### 复杂度分析
这里直接引用一下[kaiwensun的回答](https://leetcode.com/problems/critical-connections-in-a-network/discuss/382638/No-TarjanDFS-detailed-explanation-O)。
> 
DFS time complexity is O(E+V), attempting to visit each edge at most twice. (the second attempt will immediately return.)  
As the graph is always a connected graph, E >= V.  
So, time complexity = O(E).  
Space complexity = O(graph) + O(rank) + O(connections) = 3 * O(E + V) = O(E).

### 小结
Graph的问题果然很巧（bian）妙（tai），下面应该复习一下Tarjan的模版题。
