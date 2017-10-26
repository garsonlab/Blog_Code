---
title: XML 解析生成工具
date: 2017-3-1
tags:
- xml解析
categories: UnityScript
---

轻量级的XML解析生成工具，具体使用如注释。

``` csharp

using System;
using System.Collections;
using System.Collections.Generic;
using System.Text;
using UnityEngine;

/*----------------------------------------------------------------
// Copyright (C) 2017
//
// 模块名：轻量级XML工具
// 创建者：Cheng
// 修改者列表：
// 创建日期：2/28/2017
// 模块描述：
//----------------------------------------------------------------*/
namespace Garson
{
    public class XML
    {
        private XmlNode root;
        public XML() { }

        public XML(string xml)
        {
            Parse(xml);
        }

        /// <summary>
        /// 从根节点获取Element
        /// </summary>
        /// <param name="path">路径，eg:"Root/Node/Element"</param>
        /// <returns></returns>
        public XmlNode GetElement(string path)
        {
            string[] paths = path.Split('/');
            var p = root;
            for (int i = 0; i < paths.Length; i++)
            {
                p = p.GetElement(paths[i]);
                if(p == null)
                    break;
            }
            return p;
        }

        /// <summary>
        /// 在指定节点下插入新元素
        /// </summary>
        /// <param name="path">如果path为空则插入在根节点</param>
        /// <param name="name">新节点名称</param>
        /// <returns></returns>
        public XmlNode AddNode(string path, string name)
        {
            if(root == null)
                root = new XmlNode();
            if (string.IsNullOrEmpty(path))
            {
                XmlNode node = new XmlNode {name = name};
                root.AddChild(node);
                return node;
            }
            else
            {
                var parent = GetElement(path);
                if (parent == null)
                {
                    Debug.LogError("Error: Cannot find path:" + path);
                    return null;
                }
                XmlNode node = new XmlNode();
                node.name = name;
                parent.AddChild(node);
                return node;
            }
        }

        /// <summary>
        /// 在指定路径下插入新节点
        /// </summary>
        /// <param name="path">如果路径不存在，会创建相应的节点</param>
        /// <param name="name">新节点名称</param>
        /// <returns></returns>
        public XmlNode AddNodeIgnorePath(string path, string name)
        {
            if (root == null)
                root = new XmlNode();

            string[] paths = path.Split('/');
            var p = root;
            for (int i = 0; i < paths.Length; i++)
            {
                var c = p.GetElement(paths[i]);
                if (p == null)
                {
                    c = new XmlNode(){name = paths[i]};
                    p.AddChild(c);
                }
                p = c;
            }

            var node = new XmlNode(){name = name};
            p.AddChild(node);
            return node;
        }

        /// <summary>
        /// 添加属性
        /// </summary>
        /// <param name="path">该路径下的节点</param>
        /// <param name="key">属性名</param>
        /// <param name="value">属性值</param>
        /// <returns></returns>
        public bool AddAttribute(string path, string key, string value)
        {
            var p = GetElement(path);
            if (p == null) return false;
            p.AddAttribute(key, value);
            return true;
        }

        public override string ToString()
        {
            return Print();
        }

        /// <summary>
        /// 把XML转化成字符串
        /// </summary>
        /// <returns></returns>
        public string Print()
        {
            StringBuilder stringBuilder = new StringBuilder();
            stringBuilder.AppendLine("<?xml version=\"1.0\" encoding=\"UTF-8\"?>");

            var elements = root.GetElements();
            foreach (var element in elements)
            {
                BuildString(stringBuilder, element, 0);
            }
            return stringBuilder.ToString();
        }

        /// <summary>
        /// 递归调用
        /// </summary>
        /// <param name="stringBuilder"></param>
        /// <param name="element"></param>
        /// <param name="tab">每行前加制表符个数</param>
        private void BuildString(StringBuilder stringBuilder, XmlNode element, int tab)
        {
            for (int i = 0; i < tab; i++)
            {
                stringBuilder.Append("\t");
            }
            stringBuilder.Append("<");
            stringBuilder.Append(element.name);
            if (element.HasAttribute())
            {
                stringBuilder.Append(" ");
                var attrs = element.GetAttributes();
                foreach (var key in attrs.Keys)
                {
                    stringBuilder.Append(key);
                    stringBuilder.Append("=\"");
                    stringBuilder.Append(attrs[key]);
                    stringBuilder.Append("\" ");
                }
            }
            if (element.HasChild())
            {
                stringBuilder.AppendLine(">");
                var childern = element.GetElements();
                foreach (var child in childern)
                {
                    BuildString(stringBuilder, child, tab+1);
                }
                for (int i = 0; i < tab; i++)
                {
                    stringBuilder.Append("\t");
                }
                stringBuilder.AppendLine("</" + element.name + ">");
            }
            else if (!string.IsNullOrEmpty(element.text))
            {
                stringBuilder.Append(">");
                stringBuilder.Append(element.text);
                stringBuilder.AppendLine("</" + element.name + ">");
            }
            else
            {
                stringBuilder.AppendLine(" />");
            }
        }

        private const char LT = '<';
        private const char GT = '>';
        private const char DASH = '-';
        private const char SPACE = ' ';
        private const char QUOTE = '"';
        private const char SLASH = '/';
        private const char QMARK = '?';
        private const char EQUALS = '=';
        private const char EXCLAMATION = '!';

        private enum ElementType
        {
            /// <summary>
            /// 元标签
            /// </summary>
            METATAG,
            /// <summary>
            /// 注释
            /// </summary>
            COMMENT,
            /// <summary>
            /// 声明
            /// </summary>
            DOCTYPE,
            /// <summary>
            /// 字符数据
            /// </summary>
            CDATA,
            /// <summary>
            /// 空
            /// </summary>
            NONE,
        }

        /// <summary>
        /// Xml解析，会覆盖已存在的XML
        /// </summary>
        /// <param name="xml"></param>
        public void Parse(string xml)
        {
            if(string.IsNullOrEmpty(xml))
                return;
            xml = xml.Replace("\n", " ");
            root = new XmlNode();

            ElementType cType = ElementType.NONE;
            
            char c; //current char
            char cp; //c previous
            char cn; //c next
            char cnn; // c next next
            Stack<XmlNode> xmlStack = new Stack<XmlNode>();
            xmlStack.Push(root);
            int length = xml.Length;
            bool collectingName = false;
            bool collectingAttribute = false;
            string name = "";
            string attr = "";

            for (int i = 0; i < length; i++)
            {
                c = cp = cn = cnn = '\0';
                c = xml[i];
                if (i > 0) cp = xml[i - 1];
                if (i + 1 < length) cn = xml[i + 1];
                if (i + 2 < length) cnn = xml[i + 2];

                if (cType == ElementType.NONE)
                {
                    if (c == LT)
                    {
                        if (xmlStack.Count > 0)
                        {
                            if (!string.IsNullOrEmpty(name.Trim()))// top ele's text eg.<aa>****</aa>
                            {
                                var node = xmlStack.Peek();
                                node.text = name.Trim();
                            }
                        }
                        name = "";

                        if (cn == QMARK)//<?*****?>
                        {
                            cType = ElementType.METATAG;
                            i++;
                            continue;
                        }
                        if (cn == EXCLAMATION && cnn == DASH)//<!-->
                        {
                            cType = ElementType.COMMENT;
                            i++;
                            continue;
                        }
                        if (cn == EXCLAMATION)//<![[*******>
                        {
                            cType = ElementType.DOCTYPE;
                            i++;
                            continue;
                        }

                        if (cn == SLASH)//</***>
                        {
                            cType = ElementType.CDATA;
                            continue;
                        }
                        //create new
                        cType = ElementType.CDATA;
                        collectingName = false;
                        collectingAttribute = false;
                        name = "";
                        attr = "";
                        XmlNode xmlNode = new XmlNode();
                        if (xmlStack.Count > 0)
                        {
                            var parent = xmlStack.Peek();
                            parent.AddChild(xmlNode);
                        }
                        xmlStack.Push(xmlNode);
                        continue;
                    }
                    else
                    {
                        name += c;
                    }
                }

                if (cType == ElementType.METATAG)
                {
                    if (c == QMARK && cn == GT)
                    {
                        i++;
                        cType = ElementType.NONE;
                        continue;
                    }
                    else
                        continue;
                }
                if (cType == ElementType.COMMENT)
                {
                    if (c == DASH && cn == DASH && cnn == GT)
                    {
                        i += 2;
                        cType = ElementType.NONE;
                        continue;
                    }
                    else
                        continue;
                }
                if (cType == ElementType.DOCTYPE)
                {
                    if (c == GT) cType = ElementType.NONE;
                    continue;
                }

                if (cType == ElementType.CDATA)
                {
                    if (collectingName)
                    {
                        if (c == SPACE || c == GT)
                        {
                            var node = xmlStack.Peek();
                            if (string.IsNullOrEmpty(node.name))
                            {
                                node.name = name.Trim();
                                collectingName = false;
                                name = "";
                            }
                            if (c == GT)
                            {
                                cType = ElementType.NONE;
                                continue;
                            }
                            
                        }
                        else if (c == SLASH && cn == GT)
                        {
                            var node = xmlStack.Peek();
                            node.name = name.Trim();
                            collectingName = false;
                            cType = ElementType.NONE;
                            name = "";
                            i++;

                            xmlStack.Pop();
                            continue;
                        }
                        else if (c == EQUALS)
                        {
                            if (cn == QUOTE)
                            {
                                collectingName = false;
                                collectingAttribute = true;
                                attr = "";
                                i++;
                                continue;
                            }
                            else
                                Debug.LogError("Error: Attribute '\"' is not near '=' in char index:" + i );
                        }
                        else
                        {
                            name += c;
                        }
                    }
                    else if(collectingAttribute)
                    {
                        if (c == QUOTE)
                        {
                            collectingAttribute = false;
                            var node = xmlStack.Peek();
                            node.AddAttribute(name.Trim(), attr);
                            name = "";
                            attr = "";
                        }
                        else
                        {
                            attr += c;
                        }
                    }
                    else
                    {
                        if(c == SPACE) continue;

                        if (c == GT)
                        {
                            cType = ElementType.NONE;
                            continue;
                        }

                        if (c == SLASH && cn == GT)
                        {
                            cType = ElementType.NONE;
                            xmlStack.Pop();
                            i++;
                            continue;
                        }

                        if (c == SLASH)
                        {
                            name = "";
                            int j = i + 1;
                            for (; j < length; j++)
                            {
                                if (xml[j] == GT)
                                {
                                    break;
                                }
                                else
                                {
                                    name += xml[j];
                                }
                            }

                            var node = xmlStack.Peek();
                            if (node.name.Equals(name))
                            {
                                xmlStack.Pop();
                            }
                            else
                            {
                                Debug.LogError("Error: current is /, name is "+ name + ", but top node name is "+ node.name );
                            }
                            i = j;
                            cType = ElementType.NONE;
                            name = "";
                            continue;
                        }

                        name += c;
                        collectingName = true;
                    }
                }
            }
        }

    }


    public class XmlNode
    {
        public string text { get; set; }
        public string name { get; set; }
        private List<XmlNode> children;
        private Dictionary<string, string> attributes;

        public XmlNode()
        {
            children = new List<XmlNode>();
            attributes = new Dictionary<string, string>();
            name = string.Empty;
            text = string.Empty;
        }

        public bool HasChild()
        {
            return children.Count > 0;
        }

        public bool HasAttribute()
        {
            return attributes.Count > 0;
        }

        public void AddChild(XmlNode child)
        {
            children.Add(child);
        }

        /// <summary>
        /// 移除子节点
        /// </summary>
        /// <param name="index">从0开始的下标</param>
        /// <returns></returns>
        public XmlNode RemoveChild(int index)
        {
            if (children.Count > index)
            {
                var node = children[index];
                children.RemoveAt(index);
                return node;
            }
            return null;
        }

        /// <summary>
        /// 添加属性，已存在则会覆盖
        /// </summary>
        /// <param name="key"></param>
        /// <param name="value"></param>
        public void AddAttribute(string key, string value)
        {
            if (attributes.ContainsKey(key))
                attributes[key] = value;
            else
                attributes.Add(key, value);
        }

        public void RemoveAttribute(string key)
        {
            attributes.Remove(key);
        }

        /// <summary>
        /// 获取节点
        /// </summary>
        /// <param name="index">从0开始的下标</param>
        /// <returns></returns>
        public XmlNode GetElement(int index)
        {
            if (children.Count > index)
                return children[index];
            return null;
        }

        public XmlNode GetElement(string cname)
        {
            foreach (var child in children)
            {
                if (child.name.Equals(cname))
                    return child;
            }
            return null;
        }

        public List<XmlNode> GetElements()
        {
            return children;
        }

        public string GetAttribute(string key)
        {
            if (attributes.ContainsKey(key))
                return attributes[key];
            return string.Empty;
        }

        public Dictionary<string, string> GetAttributes()
        {
            return attributes;
        }

        public string[] GetAttributeArray(string key)
        {
            string value = GetAttribute(key);
            return value.Split(',');
        }
    }
}
```

可能会有坑，但是我自己还没遇到...