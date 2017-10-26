---
title: 扫雷 unity版
date: 2017-2-20
tags:
- 扫雷
- Unity
categories: UnityScript
---


以前看没想过扫雷的实现，昨天看到一个帖子发的扫雷，写的很恶心，所以自己就尝试了一下，直接新建一个cs脚本复制以下代码就可以了。
先看看效果
![扫雷图来源于我的csdn](http://img.blog.csdn.net/20170220105031070?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

代码：
``` csharp

using System.Collections.Generic;
using UnityEngine;

public class MineSweeper : MonoBehaviour
{
    public static MineSweeper instance;
    private GameObject prefab;
    private List<MineCube> objects; 

    public int Row = 5;
    public int Col = 5;
    public int Mine = 10;
    private int totalCount;

    void Start()
    {
        instance = this;
        CreatePrefab();
        objects = new List<MineCube>();
    }

    void Create()
    {
        //mine总数小于格子数
        if(Mine > Row*Col) {Debug.LogError("Mine's count canot more than grid's count!");return;}
        totalCount = Row*Col;
        //清除所有旧物体
        int count = objects.Count;
        for (int i = count-1; i >=0 ; i--)
        {
            objects[i].DoDestroy();
        }
        objects.Clear();
        //创建物体
        for (int i = 0; i < Row; i++)
        {
            for (int j = 0; j < Col; j++)
            {
                CreateObject(i, j);
            }
        }
        //创建mine
        CreateMines();
        //更新数据
        UpdateMines();
    }

    void OnGUI()
    {
        GUI.Label(new Rect(Screen.width * 0.5f - 100, 0, 200, 30),  "TotalBlock: " + Row * Col + " Mine: " + Mine);
        GUI.Label(new Rect(0,0,50,30),"Row" );
        string row = GUI.TextField(new Rect(50, 0, 50, 30), Row.ToString());
        if (!int.TryParse(row, out Row))
            Row = 3;
        GUI.Label(new Rect(0, 35, 50, 30), "Col");
        string col = GUI.TextField(new Rect(50, 35, 50, 30), Col.ToString());
        if (!int.TryParse(col, out Col))
            Col = 3;
        GUI.Label(new Rect(0, 70, 50, 30), "Mine");
        string mine = GUI.TextField(new Rect(50, 70, 50, 30), Mine.ToString());
        if (!int.TryParse(mine, out Mine))
            Mine = 3;
        if(GUI.Button(new Rect(10, 110, 50, 30), "Create"))
            Create();

    }

    /// <summary>
    /// 创建预制体，附加TextMesh
    /// </summary>
    void CreatePrefab()
    {
        prefab = GameObject.CreatePrimitive(PrimitiveType.Cube);
        GameObject text = new GameObject("Text");
        text.AddComponent<MeshRenderer>();
        TextMesh txtMesh = text.AddComponent<TextMesh>();
        text.transform.parent = prefab.transform;
        text.transform.localPosition = Vector3.zero;

        txtMesh.color = Color.black;
        txtMesh.anchor = TextAnchor.MiddleCenter;
        txtMesh.alignment = TextAlignment.Center;
        txtMesh.fontSize = 10;

        prefab.SetActive(false);
        prefab.AddComponent<MineCube>();
        prefab.transform.localScale = Vector3.one * 0.9f;
    }

    void CreateObject(int x, int y)
    {
        GameObject obj = Instantiate(prefab);
        obj.SetActive(true);
        MineCube mineCube = obj.GetComponent<MineCube>();
        mineCube.Set(x, y, objects.Count);
        objects.Add(mineCube);
    }

    void CreateMines()
    {
        //Mine个不重复随机数
        List<int> randoms = new List<int>();
        for (int i = 0; i < Mine; i++)
        {
            int r = Random.Range(0, objects.Count-1);
            while (randoms.Contains(r))
            {
                r = (r + 1)%objects.Count;
            }
            randoms.Add(r);
        }
        //设置mine
        for (int i = 0; i < Mine; i++)
        {
            objects[randoms[i]].SetState(1);
        }
    }

    /// <summary>
    /// （该部分可优化）
    /// 如果是mine，跳过
    /// 否则，判断周围是否有mine
    ///         是：直接统计九宫格内mine 个数
    ///         否：把周围空格标记
    /// </summary>
    void UpdateMines()
    {
        for (int i = 0; i < Row; i++)
        {
            for (int j = 0; j < Col; j++)
            {
                MineCube mine = objects[i*Col + j];
                if(mine.isMine()) continue;
                for (int k = i-1; k <= i+1; k++)
                {
                    for (int l = j-1; l <= j+1; l++)
                    {
                        if(k < 0 || k >= Row || l < 0 || l >= Col) continue;//划定九宫格界限
                        if(k == i && l == j) continue;//排除自身
                        if (objects[k*Col + l].isMine())
                            mine.AddAroundMine();
                        else
                            mine.AddAroundBlock(objects[k*Col + l]);
                    }
                }
            }
        }
    }

    internal void GameOver()
    {
        Debug.Log("GameOver!");
    }

    /// <summary>
    /// 点击判断，根据剩下的格子数判断输赢
    /// </summary>
    /// <param name="ismine"></param>
    internal void SendCount(bool ismine)
    {
        if (!ismine)
        {
            totalCount--;
            if (totalCount == Mine)
            {
                Debug.Log("Congratulations! You Win");
            }
        }
    }
}


public class MineCube : MonoBehaviour
{
    private TextMesh textMesh;
    private int x;
    private int y;
    private int num;
    private int state;//0空白， 1mine
    private Material mat;

    public int aroundMine;
    private List<MineCube> aroundBlock;

    public bool selected;//是否选中

    void Awake()
    {
        textMesh = GetComponentInChildren<TextMesh>();
        mat = GetComponent<Renderer>().material;
        state = 0;
        aroundMine = 0;
        aroundBlock = new List<MineCube>();
        selected = false;
    }

    public void OnMouseDown()
    {
        if(selected) return;
        selected = true;//标记选中
        MineSweeper.instance.SendCount(isMine());
        Show();
        if (isMine())//该块是mine则游戏结束
        {
            MineSweeper.instance.GameOver();
            return;
        }
    }

    /// <summary>
    /// 是mine标记X， 不是则判断周围有显示数字，没有则把相邻的所有没有的都显示出来
    /// </summary>
    private void Show()
    {
        if (isMine())
        {
            textMesh.text = "X";
            mat.SetColor("_Color", Color.red);
        }
        else
        {
            if (aroundMine != 0)
                textMesh.text = aroundMine.ToString();
            else
            {
                ShowAroundBlock();
            }
            mat.SetColor("_Color", Color.gray);
        }

    }

    /// <summary>
    /// 显示周围空白格
    /// </summary>
    public void ShowAroundBlock()
    {
        if (!selected)
        {
            this.selected = true;
            MineSweeper.instance.SendCount(false);
        }
        mat.SetColor("_Color", Color.gray);
        foreach (var _block in aroundBlock)//遍历周围的空白方格
        {
            if (!_block.selected && !_block.isMine() && _block.aroundMine == 0)//该方格未被选中且同样和当前方格一样是一个空白的，周围没有mine的
            {//点击到空白的时要一起显示空白
                _block.ShowAroundBlock();
            }
        }
    }


    /// <summary>
    /// 鼠标进入时设置颜色、因为创建的预制体有碰撞器，故OnMouse***函数有效
    /// </summary>
    public void OnMouseEnter()
    {
        mat.SetColor("_Color", Color.green);
    }

    /// <summary>
    /// 鼠标退出时恢复原状
    /// </summary>
    public void OnMouseExit()
    {
        if (selected)
        {
            if(!isMine())
                mat.SetColor("_Color", Color.gray);
            else
                mat.SetColor("_Color", Color.red);
        }
        else
            mat.SetColor("_Color", Color.white);
    }
    


    internal void Set(int x, int y, int num)
    {
        this.x = x;
        this.y = y;
        this.num = num;
        transform.position = new Vector3(x, y, 0);
    }

    internal void DoDestroy()
    {
        Destroy(gameObject);
    }

    /// <summary>
    /// state 的 0:空白没有mine
    ///          1:是mine
    /// </summary>
    /// <param name="p"></param>
    internal void SetState(int p)
    {
        state = p;
    }

    /// <summary>
    /// 判断是否是mine
    /// </summary>
    /// <returns></returns>
    internal bool isMine()
    {
        return state == 1;
    }

    /// <summary>
    /// 在UpdateMines中用来更新该block周围的Mine数量
    /// </summary>
    internal void AddAroundMine()
    {
        aroundMine++;
    }

    /// <summary>
    /// 当前格为空block时标记周围八个格中没有mine的格
    /// 用来显示连续显示
    /// </summary>
    /// <param name="block"></param>
    internal void AddAroundBlock(MineCube block)
    {
        aroundBlock.Add(block);
    }
}
```

