---
layout: post
title: "GUILayout.Label也可以支持Texture"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
---

### GUILayout.Label也可以支持Texture

之前一直不知道，GUILayout.Label函数是支持Texture的，玩 [The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr) 的时候才发现。

![img](/img/guilayout-texture/screenshot1.png){:height="60%" width="60%"}

代码好简单，如下：

```csharp
namespace IL3DN
{
    using UnityEngine;
    using UnityEditor;

    [CustomEditor(typeof(IL3DN_Wind))]
    public class IL3DN_WindEditor : Editor
    {
        SerializedProperty Wind;
        SerializedProperty WindStrenght;
        SerializedProperty WindSpeed;
        SerializedProperty WindTurbulence;
        SerializedProperty Wiggle;
        SerializedProperty LeavesWiggle;
        SerializedProperty GrassWiggle;
        Texture2D IL3DN_WindLabel;
        Texture2D IL3DN_LeavesLabel;

        void OnEnable()
        {
            Wind = serializedObject.FindProperty("Wind");
            WindStrenght = serializedObject.FindProperty("WindStrenght");
            WindSpeed = serializedObject.FindProperty("WindSpeed");
            WindTurbulence = serializedObject.FindProperty("WindTurbulence");

            Wiggle = serializedObject.FindProperty("Wiggle");
            LeavesWiggle = serializedObject.FindProperty("LeavesWiggle");
            GrassWiggle = serializedObject.FindProperty("GrassWiggle");

            IL3DN_WindLabel = AssetDatabase.LoadAssetAtPath<Texture2D>("Assets/IL3DN/EditorImages/IL3DN_Label_Wind_VertexAnimations.png");
            IL3DN_LeavesLabel = AssetDatabase.LoadAssetAtPath<Texture2D>("Assets/IL3DN/EditorImages/IL3DN_Label_Wind_UVAnimations.png");
        }

        public override void OnInspectorGUI()
        {
            GUILayout.BeginHorizontal();
            GUILayout.FlexibleSpace();
            GUILayout.Label(IL3DN_WindLabel);
            GUILayout.FlexibleSpace();
            GUILayout.EndHorizontal();

            EditorGUILayout.BeginVertical(EditorStyles.helpBox);
            EditorGUILayout.Space();
            EditorGUILayout.PropertyField(Wind, new GUIContent("Wind"));
            EditorGUILayout.PropertyField(WindStrenght, new GUIContent("Wind Strength"));
            EditorGUILayout.PropertyField(WindSpeed, new GUIContent("Wind Speed"));
            EditorGUILayout.PropertyField(WindTurbulence, new GUIContent("Wind Turbulence"));
            EditorGUILayout.Space();
            EditorGUILayout.EndVertical();

            GUILayout.BeginHorizontal();
            GUILayout.FlexibleSpace();
            GUILayout.Label(IL3DN_LeavesLabel);
            GUILayout.FlexibleSpace();
            GUILayout.EndHorizontal();

            EditorGUILayout.BeginVertical(EditorStyles.helpBox);
            EditorGUILayout.Space();
            EditorGUILayout.PropertyField(Wiggle, new GUIContent("Wiggle"));
            EditorGUILayout.PropertyField(LeavesWiggle, new GUIContent("Leaves Wiggle"));
            EditorGUILayout.PropertyField(GrassWiggle, new GUIContent("Grass Wiggle"));
            EditorGUILayout.Space();
            EditorGUILayout.EndVertical();

            serializedObject.ApplyModifiedProperties();
        }
    }
}

```
