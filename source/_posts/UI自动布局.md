---
title: UI自动布局
date: 2017-11-4
tags:
- UGUI
- Layout
categories: UGUI
---


UGUI自动布局一直适应 *GridLayoutGroup*，但是，使用grid时有一个很严重的问题就是：<font color="red">当打开**Profiler**中的*Deep Profile*时，整个Unity都会崩溃.</font><font color="green" size="1">(目前使用版本v5.6.0)</font>此外，还有一个问题，但是我给忘了...
所以，还是自己动手造轮子吧，代码如下：

``` csharp
using UnityEngine;

/// <summary>
/// Introduction: UILayout, UI自动布局，替代GridLayoutGroup, 子节点改变时需要自行调用 Rebuild
/// Author: 	Garson
/// Time: 
/// </summary>
[RequireComponent(typeof(RectTransform))]
public class UILayout : MonoBehaviour
{
    [SerializeField]//边距
    private RectOffset m_Padding = new RectOffset();
    public RectOffset padding { get { return m_Padding; } set { SetProperty(ref m_Padding, value); } }
    [SerializeField]//单元格大小
    private Vector2 m_CellSize = new Vector2(100, 100);
    public Vector2 cellSize { get { return m_CellSize; } set { SetProperty(ref m_CellSize, value);} }
    [SerializeField]//开始排布位置
    private StartCorner m_StartCorner;
    public StartCorner startCorner { get { return m_StartCorner; } set { SetProperty(ref m_StartCorner, value);} }
    [SerializeField]//排列方式
    private Arrangement m_Arrangement;
    public Arrangement arrangement { get { return m_Arrangement; } set { SetProperty(ref m_Arrangement, value);} }
    [Tooltip("<=0自动计算")]
    [SerializeField]//每行数目，<=0自动计算
    private int m_FixedCount;
    public int fixedCount { get { return m_FixedCount; } set { SetProperty(ref m_FixedCount, value); } }
    [SerializeField]
    private Vector2 m_Space;//行列间距
    public Vector2 space { get { return m_Space; } set { SetProperty(ref m_Space, value); } }
    private Vector2 m_WrapSize;//排布后该组件应该大小
    public Vector2 wrapSize { get { return m_WrapSize; } }

    private RectTransform m_rectTransform;
    private RectTransform rectTransform { get { if(m_rectTransform==null)m_rectTransform=transform as RectTransform;return m_rectTransform;} }

    public void Rebuild()
    {
        RectTransform.Edge edgeX = RectTransform.Edge.Left, edgeY= RectTransform.Edge.Top;
        float paddingX = 0, paddingY = 0, sizeX = 0, sizeY = 0;
        switch (m_StartCorner)
        {
            case StartCorner.TopLeft:
                edgeY = RectTransform.Edge.Top;
                edgeX = RectTransform.Edge.Left;
                paddingX = m_Padding.left;
                paddingY = m_Padding.top;
                sizeX = m_Padding.right;
                sizeY = m_Padding.bottom;
                break;
            case StartCorner.TopRight:
                edgeY = RectTransform.Edge.Top;
                edgeX = RectTransform.Edge.Right;                  
                paddingX = m_Padding.right;
                paddingY = m_Padding.top;
                sizeX = m_Padding.left;
                sizeY = m_Padding.bottom;
                break;
            case StartCorner.BottomLeft:
                edgeY = RectTransform.Edge.Bottom;
                edgeX = RectTransform.Edge.Left;
                paddingX = m_Padding.left;
                paddingY = m_Padding.bottom;
                sizeX = m_Padding.right;
                sizeY = m_Padding.top;
                break;
            case StartCorner.BottomRight:
                edgeY = RectTransform.Edge.Bottom;
                edgeX = RectTransform.Edge.Right;
                paddingX = m_Padding.right;
                paddingY = m_Padding.bottom;
                sizeX = m_Padding.left;
                sizeY = m_Padding.top;
                break;
        }

        int axis = 0;
        int preferCount = m_FixedCount;
        float tem = 0;
        m_WrapSize = Vector2.zero;
        switch (m_Arrangement)
        {
            case Arrangement.HorizontalFlow:
                axis = 0;
                tem = paddingX;
                if (m_FixedCount <= 0)
                {
                    preferCount = Mathf.FloorToInt(rectTransform.rect.size.x / m_CellSize.x);
                    preferCount = preferCount > 0 ? preferCount : 1;
                }
                break;
            case Arrangement.VerticalFlow:
                axis = 1;
                tem = paddingY;
                if (m_FixedCount <= 0)
                {
                    preferCount = Mathf.FloorToInt(rectTransform.rect.size.y / m_CellSize.y);
                    preferCount = preferCount > 0 ? preferCount : 1;
                }
                break;
            case Arrangement.Horizontal:
                axis = 0;
                preferCount = int.MaxValue;
                tem = paddingX;
                break;
            case Arrangement.Vertical:
                axis = 1;
                preferCount = int.MaxValue;
                tem = paddingY;
                break;
        }

        int flag = 0;
        for (int i = 0; i < transform.childCount; i++)
        {
            if (flag >= preferCount)
            {
                if (axis == 0)
                {
                    tem = paddingX;
                    paddingY += m_Space.y + m_CellSize.y;
                }
                else
                {
                    tem = paddingY;
                    paddingX += m_Space.x + m_CellSize.x;
                }
                flag = 0;
            }
            var child = transform.GetChild(i) as RectTransform;
            if (child != null && child.gameObject.activeSelf)
            {
                if (axis == 0)
                {
                    child.SetInsetAndSizeFromParentEdge(edgeX, tem, m_CellSize.x);
                    child.SetInsetAndSizeFromParentEdge(edgeY, paddingY, m_CellSize.y);
                    tem += m_Space.x + m_CellSize.x;
                    if (tem - m_Space.x > m_WrapSize.x)
                        m_WrapSize.x = tem - m_Space.x;
                    m_WrapSize.y = paddingY + m_CellSize.y;
                }
                else
                {
                    child.SetInsetAndSizeFromParentEdge(edgeX, paddingX, m_CellSize.x);
                    child.SetInsetAndSizeFromParentEdge(edgeY, tem, m_CellSize.y);
                    tem += m_Space.y + m_CellSize.y;

                    if (tem - m_Space.y > m_WrapSize.y)
                        m_WrapSize.y = tem - m_Space.y;
                    m_WrapSize.x = paddingX + m_CellSize.x;
                }
                flag++;
            }
        }
        m_WrapSize.x += sizeX;
        m_WrapSize.y += sizeY;
    }

    public void SetPadding(int left, int right, int top, int bottom)
    {
        padding = new RectOffset(left, right, top, bottom);
    }

    public void SetStartCorner(int start)
    {
        startCorner = (StartCorner) start;
    }

    public void SetArrangement(int arrange)
    {
        arrangement = (Arrangement) arrange;
    }


    private void SetProperty<T>(ref T currentValue, T newValue)
    {
        if ((currentValue == null && newValue == null) || (currentValue != null && currentValue.Equals(newValue)))
            return;
        currentValue = newValue;
        Rebuild();
    }

    [LuaInterface.NoToLua]
    public enum StartCorner
    {
        TopLeft = 0,
        TopRight = 1,
        BottomLeft = 2,
        BottomRight = 3
    }

    [LuaInterface.NoToLua]
    public enum Arrangement
    {
        //横向流动，第一排、第二排...
        HorizontalFlow = 0,
        //纵向流动，第一列、第二列...
        VerticalFlow = 1,
        //横排，相当于fixedCount=1效果
        Horizontal = 2,
        //纵排，相当于fixedCount=1效果
        Vertical = 3,
    }

}




#if UNITY_EDITOR

[UnityEditor.CustomEditor(typeof(UILayout))]
public class LayoutEditor : UnityEditor.Editor
{
    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        GUILayout.Space(10);
        if (GUILayout.Button("Rebuild"))
        {
            UILayout layout = target as UILayout;
            layout.Rebuild();
        }
    }
}
#endif
```


其中 *SetInsetAndSizeFromParentEdge* 是调用的系统方法，作用是 **设置组件相对于父节点的位置Edge距离（参数1）且在该轴上的长度（参数2）**

<font color="red">注：任何子节点的改变都不会像gridlayoutgroup一样刷新，需要手动调用Rebuild，因为觉得没有必要继承*UIBehaviour*而是继承了*MonoBehaviour*。   设置属性会自动刷新.</font>

``` bash
wrapSize 是rebuild后整个包裹区域应该的大小，可用于重新设置尺寸
```