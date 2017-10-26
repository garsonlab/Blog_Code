---
title: 关于A*的构想
date: 2017-2-13
tags:
- A*
- 优化
categories: UnityScript
---

### 原理及伪代码实现
A Star 算法的具体作用可以忽略不表了，基本上想用的都知道，不知道的基本上不在乎。
具体伪代码如下： 
``` csharp
void FindPath(Point[,] maps, Point start, Point end)  
{  
    openList.Clear();//开启列表，就是一个等待检查方格的列表  
    closeList.Clear();//闭合列表，不需要再次检查的方格  

    openList.Add(start);//添加起始点至待检查  
    bool isFound = false;  

    while (openList.Count > 0)  
    {  
        //通过F=G+H寻找开启列表中最靠谱节点  
        var temStart = GetMinCostInOpen();  
        openList.Remove(temStart);//找到后加入闭合列表  
        closeList.Add(temStart);  

        //寻找该节点周围节点,除不可访问节点和已访问节点  
        var neighbors = GetNeighbors(temStart);  
        foreach (var neighbor in neighbors)  
        {  
            if (openList.Contains(neighbor))  
            {  
                //如果开启列表中已有该节点，通过比较到起始点距离（因为  
                    //到结束点距离不会改变）  
                int c = neighbor.TryGetCostToStart(temStart);  
                if (c < neighbor.g)  
                {  
                    //如果小于原花费，则更新花费，并重设父节点为当前节点  
                    neighbor.SetParent(temStart);  
                }  
            }  
            else  
            {  
                //如果开启列表不包含该节点，则加入并设置父节点  
                openList.Add(neighbor);  
                neighbor.SetParent(temStart);  
            }  
        }  

        if (openList.Contains(end))  
        {  
            //如果开启列表中已包含结束点则证明找到  
            isFound = true;  
            break;  
        }  
    }  

    //通过isFound判断是否找到，已找到后可以根据结束点父节点倒推路劲  
} 
```

至于具体的原理及意义可参考：[理解AStar寻路算法具体过程](http://www.cnblogs.com/technology/archive/2011/05/26/2058842.html)


### 优化构想
Astar作为寻路算法应用十分广泛，但是，如果在一个超大的地图上、多人同时寻路中，超长距离的寻路对性能的消耗十分严重，因为它需要遍历格子，还需要维持开启列表找到最小值，So， 关于Astar的优化，个人做了两点考虑，
1，缩小寻路块
2，更方便的维持开启列表
对于第二点，个人的考虑是，在搜索到周围节点加入开启列表时，维持开启列表从小到大的有序性。具体做法是用一种算法控制查找等的消耗时间，比如，用二叉树控制插入的节点，使父节点永远小于子节点，或者直接使用折半查找等，怎么高兴怎么来，最后看疗效决定。
对于第一点，个人目前的的做法是使用 四叉树 进行场景管理。首先在处理完地图以后，进行四叉树的构建和场景合并：某一层次下叶节点全可通过则标记该节点可通过，反之亦然。在寻路的过程中，首先确定起始点所在的节点，通过节点进行寻路，具体做法同Astar。这样，假如我们就算最底层的长度为2，我们也相当于把地图缩小了四倍。但有一点，我们创建四叉树时，节点的坐标一般选择所表示的区域中心点，所以用来做寻路的也是该区域的中心点。也就是或出现具体执行路径时是这种情况：玩家从当前所在位置——>玩家所在节点的中心点——>正常寻路——>结束点所在节点的中心点——>结束点。开始和结束都要去节点中心点，这个在短距离寻路中，会有绕行的感觉，所以，不适合短距离寻路...T.T
个人的 实际代码未整理有些凌乱暂时就不贴了，效率自测，应该不会辜负你的期望。