```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.InputSystem; // 【必须添加】引入新版输入系统的命名空间

public class Ripple : MonoBehaviour
{
    public Camera mainCamera;
    public RenderTexture DrawRT; 
    public RenderTexture TempRT;
    public Shader DrawShader;

    private Material DrawMat;
    
    public int TextureSize = 512; 

    void Start()
    {
        mainCamera = Camera.main;
        
        DrawRT = CreateRT();
        TempRT = CreateRT();

        DrawMat = new Material(DrawShader);
        GetComponent<Renderer>().material.mainTexture = DrawRT;
    }

    public RenderTexture CreateRT() 
    {
        RenderTexture rt = new RenderTexture(TextureSize, TextureSize, 0, RenderTextureFormat.RFloat);
        rt.Create();
        return rt;
    }

    private void DrawAt(float x, float y, float radius)
    {
        DrawMat.SetTexture("_SourceTex", DrawRT);
        DrawMat.SetVector("_Pos", new Vector4(x, y, radius));
        Graphics.Blit(null, TempRT, DrawMat);

        RenderTexture rt = TempRT;
        TempRT = DrawRT;
        DrawRT = rt;
    }

    void Update()
    {
        // 【修改】使用新版 Input System 的鼠标检测方式
        if (Mouse.current != null && Mouse.current.leftButton.isPressed)
        {
            // 获取新版系统下的鼠标屏幕坐标
            Vector2 mousePos = Mouse.current.position.ReadValue();
            
            Ray ray = mainCamera.ScreenPointToRay(mousePos);
            RaycastHit hit; 
            
            if (Physics.Raycast(ray, out hit)) 
            {
                DrawAt(hit.textureCoord.x, hit.textureCoord.y, 0.1f);
            }
        }
    }
}
```