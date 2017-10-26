---
title: 踩坑填坑——DropDown
date: 2017-8-8
tags:
- UGUI
- Dropdown
categories: UGUI
---

在使用UGUI的 DropDown 时， Canvas 的 Render Mode 选择了 Screen Space--Camera， 此时遇到一个小bug， 当我把这个下拉组件放到屏幕中间附近时， 下拉列表显示是正常的。当我把组件整体移到边缘，突然出现下拉列表的 Content 的坐标 不合法，由于 ugui 的点击关闭处理是在 Canvas 的子节点最下方又生成一个 全屏的 遮罩 来保证实现 "点击关闭"，所以此时整个界面卡死..........翻遍源码断点我也没找到问题..


![我的csdn](http://img.blog.csdn.net/20170808122348361?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)（中间：边缘）![我的csdn](http://img.blog.csdn.net/20170808122416928?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![我的csdn](http://img.blog.csdn.net/20170808122431717?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

另外，他的下拉列表实现， 不适合多个数据，假如有百十个就生成百十个下来的子菜单，这明显是不合理的，所以，把下拉改成 无限循环 列表是必须的。
于是，开始自己动手造轮子：
``` csharp

using System;
using System.Collections;
using System.Collections.Generic;
using Assets.UI;
using UnityEngine;
using UnityEngine.UI;

/// <summary>
/// Introduction: GDropDown
/// Author:     Cheng
/// Time: 
/// </summary>
[AddComponentMenu("UI/GDropdown", 100)]
[RequireComponent(typeof(RectTransform))]
public class GDropDown : MonoBehaviour
{

    [Tooltip("Button of Whole Component")]
    [SerializeField]
    private Toggle m_CaptionToggle;
    /// <summary>
    /// Button of Whole Component
    /// </summary>
    public Toggle CaptionToggle { get { return m_CaptionToggle; } set { SetCaptionButton(value);} }

    [Tooltip("Display Text of Selected Item")]
    [SerializeField]
    private Text m_CaptionText;
    /// <summary>
    /// Display Text of Selected Item
    /// </summary>
    public Text CaptionText { get { return m_CaptionText; } set { m_CaptionText = value; } }

    [Tooltip("Display Image of Selected Item")]
    [SerializeField]
    private Image m_CaptionImage;
    /// <summary>
    /// Display Image of Selected Item
    /// </summary>
    public Image CaptionImage { get { return m_CaptionImage; } set { m_CaptionImage = value; } }

    [Space]

    [Tooltip("Drop List")]
    [SerializeField]
    private ScrollRect m_ScrollRect;
    /// <summary>
    /// Drop List 
    /// </summary>
    public ScrollRect ScrollRect { get { return m_ScrollRect; } set { m_ScrollRect = value; } }

    [Tooltip("Template of Drop List's Item")]   
    [SerializeField]
    private GameObject m_DropItem;
    /// <summary>
    /// Template of Drop List's Item
    /// </summary>
    public GameObject DropItem { get { return m_DropItem; } set { SetDropItem(value); } }

    [Tooltip("Current Select Index")]
    [SerializeField]
    private int m_Index;
    /// <summary>
    /// Current Select Index
    /// </summary>
    public int Index { get { return m_Index; } set { SetSelectIndex(value); }}
    public string Text { get { return m_DropData.Count > Index ? m_DropData[Index].text : ""; } }

    /// <summary>
    /// Drop Down Value Changed
    /// </summary>
    public Callback_1<int> OnValueChanged;

    [Tooltip("Drop Data")]
    [SerializeField]
    List<GItemData> m_DropData = new List<GItemData>();

    private ScrollList m_ScrollList;
    private Dictionary<Transform, GItem> m_Items = new Dictionary<Transform, GItem>();
    private RectTransform m_PointerMask;
    private Transform m_Canvas;
    

    void Awake()
    {
        m_ScrollList = GetOrAddComponent<ScrollList>(ScrollRect.gameObject);
        m_ScrollList.onItemRender = OnItemRender;
        SetSelectIndex(m_Index);
        SetDropItem(m_DropItem);
        RefreshShowValue();
        SetCaptionButton(m_CaptionToggle);

        m_CaptionToggle.isOn = false;
        CloseMask();
    }

    public void AddOptions(string[] options)
    {
        for (int i = 0; i < options.Length; i++)
            this.m_DropData.Add(new GItemData(options[i]));

        if (m_CaptionToggle.isOn)
            OpenMask();
    }

    public void AddOptions(Sprite[] options)
    {
        for (int i = 0; i < options.Length; i++)
            this.m_DropData.Add(new GItemData(options[i]));

        if (m_CaptionToggle.isOn)
            OpenMask();
    }

    public void AddOptions(GItemData[] options)
    {
        for (int i = 0; i < options.Length; i++)
            this.m_DropData.Add(options[i]);

        if (m_CaptionToggle.isOn)
            OpenMask();
    }

    public void RemoveAt(int index)
    {
        if (this.m_DropData.Count > index)
        {
            this.m_DropData.RemoveAt(index);
            if (index == Index)
            {
                SetSelectIndex(index-1);
            }
        }
    }

    public void ClearOptions()
    {
        this.m_DropData.Clear();
        if (m_CaptionToggle.isOn)
            OpenMask();
    }


    /// <summary>
    /// Refresh Display View
    /// </summary>
    private void RefreshShowValue()
    {
        if (m_DropData.Count > Index)
        {
            GItemData data = m_DropData[Index];
            if (CaptionText != null)
                CaptionText.text = data.text;
            if (CaptionImage != null)
                CaptionImage.sprite = data.image;
        }
        else
        {
            if (CaptionText != null)
                CaptionText.text = "";
            if (CaptionImage != null)
                CaptionImage.sprite = null;
        }
    }

    /// <summary>
    /// Render Item in List
    /// </summary>
    /// <param name="index"></param>
    /// <param name="child"></param>
    private void OnItemRender(int index, Transform child)
    {
        GItem item;
        if (!m_Items.TryGetValue(child, out item))
            item = new GItem(this, child);

        if (m_DropData.Count > index)
        {
            item.Reset(m_DropData[index].text, m_DropData[index].image, index == Index);
        }
    }

    /// <summary>
    /// Set Cur Select Index When Click
    /// </summary>
    /// <param name="p"></param>
    internal void SetSelectIndex(int p)
    {
        if (p < m_DropData.Count) //if exist data
        {
            m_Index = p;
            RefreshShowValue();
            if (OnValueChanged != null)
            {
                OnValueChanged(m_Index);
            }
        }
        else
        {
            if (m_DropData.Count > 0)//Back To First
            {
                m_Index = 0;
                RefreshShowValue();
                if (OnValueChanged != null)
                {
                    OnValueChanged(m_Index);
                }
            }
        }

        foreach (var value in m_Items.Values)
        {
            value.SetActive(false);
        }
        m_CaptionToggle.isOn = false;
    }

    /// <summary>
    /// Open Mask to Poniters Out of List
    /// </summary>
    private void OpenMask()
    {
        if (m_PointerMask == null)//Create Mask
        {
            GameObject o = new GameObject("Pointer Mask");
            o.transform.SetParent(transform);
            Image mask = o.AddComponent<Image>();
            mask.color = new Color(1, 1, 1, 0);
            m_PointerMask = o.transform as RectTransform;
            m_PointerMask.sizeDelta = new Vector2(Screen.width, Screen.height);
            Button btnMask = o.AddComponent<Button>();
            btnMask.onClick.AddListener(CloseMask);
        }

        if (m_Canvas == null)//Find Canvas
        {
            Canvas canvas = GameObject.FindObjectOfType<Canvas>();
            m_Canvas = canvas.transform;
        }

        m_PointerMask.gameObject.SetActive(true);
        m_PointerMask.SetParent(m_Canvas);
        m_PointerMask.localPosition = Vector3.zero;

        if (m_ScrollRect != null)
        {
            m_ScrollRect.transform.SetParent(m_Canvas);
            m_ScrollRect.gameObject.SetActive(true);
            m_ScrollList.ChildCount = m_DropData.Count;
        }
    }

    /// <summary>
    /// Close Mask to Other Pointers
    /// </summary>
    private void CloseMask()
    {
        if (m_PointerMask != null)
        {
            m_PointerMask.transform.SetParent(transform);
            m_PointerMask.gameObject.SetActive(false);
        }

        if (m_ScrollRect != null)
        {
            m_ScrollRect.transform.SetParent(transform);
            m_ScrollRect.gameObject.SetActive(false);
        }
    }

    /// <summary>
    /// Set Drop Item, Set Anchor Left-Top
    /// </summary>
    /// <param name="item"></param>
    private void SetDropItem(GameObject item)
    {
        RectTransform rect = item.transform as RectTransform;
        rect.anchorMin = Vector2.up;
        rect.anchorMax = Vector2.up;
        m_DropItem = item;
        m_ScrollList.Child = item;
    }

    /// <summary>
    /// Set Outter Button of Whole Component
    /// </summary>
    /// <param name="btn"></param>
    private void SetCaptionButton(Toggle btn)
    {
        if (m_CaptionToggle != null)
            m_CaptionToggle.onValueChanged.RemoveListener(OnCaptionButtonClicked);

        m_CaptionToggle = btn;
        if(m_CaptionToggle != null)
            m_CaptionToggle.onValueChanged.AddListener(OnCaptionButtonClicked);
    }

    /// <summary>
    /// On Caption Button Clicked
    /// </summary>
    private void OnCaptionButtonClicked(bool active)
    {
        if (active)
            OpenMask();
        else
            CloseMask();
    }

    /// <summary>
    /// Get Or Add Component on O
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="o"></param>
    /// <returns></returns>
    T GetOrAddComponent<T>(GameObject o) where T : Component
    {
        T com = o.GetComponent<T>();
        if (com == null)
            com = o.AddComponent<T>();
        return com;
    }

    /// <summary>
    /// Release All
    /// </summary>
    public void OnDestroy()
    {
        //If Release on Drop State, Delete Mask
        if(m_PointerMask != null)
            m_PointerMask.SetParent(transform);
    }

    /// <summary>
    /// Renderer Item in Endless List
    /// </summary>
    protected internal class GItem
    {
        GDropDown dropDown;
        Transform item;
        Button btn;

        Text text;
        Image image;
        GameObject selected;
        bool activeSelf;

        public string m_Text { get { return text.text; } set { text.text = value; } }
        public Sprite m_Image { get { return image.sprite; } set { image.sprite = value; } }

        public GItem(GDropDown parent, Transform item)
        {
            this.dropDown = parent;
            this.item = item;
            activeSelf = false;

            Transform t_trans = item.FindChild("text");
            if (t_trans)
            {
                text = t_trans.gameObject.GetComponent<Text>();
            }
            Transform t_image = item.FindChild("image");
            if (t_image)
            {
                image = t_image.gameObject.GetComponent<Image>();
            }
            Transform t_selected = item.FindChild("selected");
            if (t_selected)
            {
                selected = t_selected.gameObject;
            }

            btn = item.GetComponent<Button>();
            if (btn == null)
            {
                Transform t_btn = item.FindChild("btn");
                if (t_btn != null)
                    btn = dropDown.GetOrAddComponent<Button>(t_btn.gameObject);
                else
                    btn = dropDown.GetOrAddComponent<Button>(item.gameObject);
            }

            btn.onClick.AddListener(OnBtnItemClicked);
        }

        private void OnBtnItemClicked()
        {
            if (!activeSelf)
            {
                SetActive(true);
                dropDown.SetSelectIndex(int.Parse(item.name));
            }
        }

        internal void SetActive(bool active)
        {
            this.activeSelf = active;
            if(selected != null)
                selected.SetActive(active);
        }


        internal void Reset(string txt, Sprite sprite, bool active)
        {
            if (text != null)
                m_Text = txt;
            if (image != null)
                m_Image = sprite;
            SetActive(active);
        }
    }

    /// <summary>
    /// Cache Data
    /// </summary>
    [Serializable]
    public class GItemData
    {
        [SerializeField]
        private string m_Text;
        [SerializeField]
        private Sprite m_Image;
        public string text  { get { return m_Text; }  set { m_Text = value; } }
        public Sprite image { get { return m_Image; } set { m_Image = value; } }
        public GItemData(){}

        public GItemData(string text)
        {
            this.text = text;
        }

        public GItemData(Sprite image)
        {
            this.image = image;
        }

        public GItemData(string text, Sprite image)
        {
            this.text = text;
            this.image = image;
        }
    }

}
```

![我的csdn](http://img.blog.csdn.net/20170808122959444?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

组件的点击由Toggle 来控制 与 系统UGUI的类似，支持 点击选中 图片和文字， 自己添加一个下拉列表。

<font color="red">注意： 下拉子选项中， 命名 “text”的“Text”组件、命名 “image”的“Image”组件为该选项的可填充值，看代码一眼便知。命名“selected”的表示下拉列表打开时该选项选中的表现，与toggle相同，我只是乐意改成了button表示而已</font>

Index 就是当前选中项,无限列表需要用到我以前文章中的一篇[【UGUI】无限列表 ScrollView List](https://garsonlab.github.io/2017/05/02/%E6%97%A0%E9%99%90%E5%88%97%E8%A1%A8%20ScrollView%20List/)


