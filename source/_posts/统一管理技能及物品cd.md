---
title: 统一管理技能及物品cd
date: 2017-3-20
tags:
- cd管理
- Unity
categories: UnityScript
---


有个需求，比如使用过物品以后， 技能后以一个cd时间，要求技能面板上和技能栏以及其他一切有可以使用该技能的地方做一个同步cd，因此有下面的管理模块。

``` csharp

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;


namespace Assets.UI
{
    /// <summary>
    /// Introduction: CD 统一管理类,实现多个cd同动
    /// Author: Cheng
    /// Time: 
    /// </summary>
    public class CDManager
    {
        private Dictionary<string, CDMask> m_cds; 
        private static CDManager instance;
        public static CDManager Instance
        {
            get
            {
                if(instance == null)
                    instance = new CDManager();
                return instance;
            }
        }

        public CDManager()
        {
            m_cds = new Dictionary<string, CDMask>();
        }

        /// <summary>
        /// 添加CD
        /// </summary>
        /// <param name="_id">标识符</param>
        /// <param name="_image">cd图片</param>
        /// <param name="_cd">cd时长</param>
        /// <returns></returns>
        public CDMask AddCD(string _id, Image _image, float _cd)
        {
            CDMask mask;
            if (!m_cds.TryGetValue(_id, out mask))
            {
                mask = new CDMask(_cd);
                m_cds.Add(_id, mask);
            }
            mask.AddMask(_image);
            return mask;
        }

        /// <summary>
        /// 移除CD
        /// </summary>
        /// <param name="_id">标识符</param>
        /// <param name="_image">cd图片</param>
        public void RemoveCD(string _id, Image _image)
        {
            CDMask mask;
            if (m_cds.TryGetValue(_id, out mask))
            {
                mask.RemoveMask(_image);
            }
        }

        /// <summary>
        /// 获取cd管理项
        /// </summary>
        /// <param name="_id"></param>
        /// <returns></returns>
        public CDMask GetCDMask(string _id)
        {
            if (m_cds.ContainsKey(_id))
                return m_cds[_id];
            return null;
        }
    }


    public class CDMask
    {
        private List<Image> m_imageList; //同一cd列表
        private float m_cd;//cd时间
        private float m_amount;//当前amount
        private bool m_inCD;//是否在cd中
        private float m_interval;//每次变化时间间隔
        private float m_perAmount;//每次变化量
        /// <summary>
        /// 填充比率
        /// </summary>
        public float fillAmount { get { return m_amount; } set { SetAmount(value); } }
        /// <summary>
        /// cd时长
        /// </summary>
        public float cd { get { return m_cd; } set { SetCD(value); } }
        /// <summary>
        /// cd开始和结束事件
        /// </summary>
        public event Callback_1<bool> cdStateChanged;

        public CDMask(float _cd, float _interval = 0.1f)
        {
            m_imageList = new List<Image>();
            m_interval = _interval;
            cd = _cd;
            m_amount = 0;
            m_inCD = false;
            fillAmount = 0;
        }

        /// <summary>
        /// 设置填充比率
        /// </summary>
        /// <param name="a">比率，范围【0-1】</param>
        public void SetAmount(float a)
        {
            m_amount = Mathf.Clamp01(a);
            for (int i = m_imageList.Count-1; i >= 0; i--)
            {
                if (m_imageList[i] != null)
                    m_imageList[i].fillAmount = m_amount;
            }

            if (m_amount > 0 && !m_inCD)//当前不在cd中且cd>0则开始cd
            {
                Scheduler.Instance.RepeatCall(m_interval, UpdateAmount);
                m_inCD = true;
                if (cdStateChanged != null)
                    cdStateChanged(m_inCD);
            }
            else if (m_amount <= 0 && m_inCD)//cd结束
            {
                Scheduler.Instance.CancelCallback(UpdateAmount);
                m_inCD = false;
                if (cdStateChanged != null)
                    cdStateChanged(m_inCD);
            }
        }

        private void UpdateAmount(float dt)
        {
            SetAmount(m_amount - m_perAmount);
        }

        /// <summary>
        /// 设置整体的cd时长
        /// </summary>
        /// <param name="_cd"></param>
        public void SetCD(float _cd)
        {
            m_cd = _cd;

            if (m_interval <= 0)
                m_perAmount = m_cd;
            else
            {
                int t = Mathf.CeilToInt(_cd / m_interval);
                m_perAmount = 1.0f / t;
            }
        }

        /// <summary>
        /// 获取当前是否在cd中
        /// </summary>
        /// <returns></returns>
        public bool GetCDState()
        {
            return m_inCD;
        }

        /// <summary>
        /// 设置Tick时长，默认0.1
        /// </summary>
        /// <param name="interval"></param>
        public void SetInterval(float interval)
        {
            m_interval = interval;
            SetCD(m_cd);
        }

        /// <summary>
        /// 添加cd
        /// </summary>
        /// <param name="_image"></param>
        public void AddMask(Image _image)
        {
            if (_image.type != Image.Type.Filled)//如果当前不是填充模式则改为顶部转圈
            {
                _image.type = Image.Type.Filled;
                _image.fillMethod = Image.FillMethod.Radial360;
                _image.fillOrigin = (int)Image.Origin360.Top;
                _image.fillClockwise = false;
            }

            if(!m_imageList.Contains(_image))
                m_imageList.Add(_image);
            _image.fillAmount = m_amount;
        }

        /// <summary>
        /// 移除cd
        /// </summary>
        /// <param name="_image"></param>
        public void RemoveMask(Image _image)
        {
            if (m_imageList.Contains(_image))
                m_imageList.Remove(_image);
        }


    }

}
```

 其中， Scheduler.Instance.RepeatCall是一个计时器回调，可以在Update里做相应的计时替换，每次更新调用的时长为m_internal。
使用方式为
``` csharp
cdItem = CDManager.Instance.AddCD("Key", images, cdtime);
```

使用相同的cd则直接把key设置相同即可，通过返回的cdItem进行其他的操作。
![图片来源我的csdn](http://img.blog.csdn.net/20170320160355897?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ2hlbmc2MjQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

得到结果所有编号1的cd转动相同，2的也相同。