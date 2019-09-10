# LWRP
LWRP Basic shader code


Shader "LWSampleShader"

{

    Properties
    {
        [Space(20)]
        [Header(LWRP Demo)]    

        _TintColor ("Color", color) = (1,1,1,1)
        _MainTex("Texture", 2D) = "white" {}
        [KeywordEnum(LC, LW1, LW2)] _Sample("Sampler mode", Float) = 0
        _Offset("Tiling and Offset", vector) = (1, 1, 0, 0)
       
        [Space(10)]
        [Header(Lighting Mode)]
        [KeywordEnum(None, Lambert, BlinnPhong)] _LI("Lighitng mode", Float) = 0

        [Space(10)]
        [Header(Alpha features)]
        _Alpha("AlphaThreshold", Range(0,1)) = 1

        [Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend("Src Blend", Float) = 1
        [Enum(UnityEngine.Rendering.BlendMode)] _DstBlend("Dst Blend", Float) = 1
        [Enum(Off, 0, On, 1)] _ZWrite("ZWrite", Float) = 0
        [Enum(UnityEngine.Rendering.CompareFunction)] _ZTest("ZTest", Float) = 0
        [Enum(UnityEngine.Rendering.CullMode)] _Cull("Cull Mode", Float) = 1 


    }
    
    SubShader
    {
      

      // Select Render type

      Tags {  "RenderPipeline"="LightweightPipeline" "RenderType"="Opaque" "Queue"="Geometry"}
      //Tags {"RenderPipeline"="LightweightPipeline" "RenderType"="TransparentCutout" "Queue"="AlphaTest"}
      //Tags {"RenderPipeline"="LightweightPipeline" "RenderType"="Transparent" "Queue"="Transparent"}

        LOD 100

        Pass
        {
                      
           Blend[_SrcBlend][_DstBlend]
           ZWrite[_ZWrite]
           ZTest[_ZTest]
           Cull[_Cull]


           HLSLPROGRAM

            #pragma prefer_hlslcc gles

            #pragma exclude_renderers d3d11_9x

           // #pragma shader_feature _SAMPLE_GI   

            #pragma vertex vert
            #pragma fragment frag

            #pragma multi_compile _SAMPLE_LC _SAMPLE_LW1 _SAMPLE_LW2
            #pragma multi_compile _LI_NONE _LI_LAMBERT _LI_BLINNPHONG

            #include "Packages/com.unity.render-pipelines.lightweight/ShaderLibrary/Lighting.hlsl"

             #if _SAMPLE_LC
            // macro include
            # include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Macros.hlsl"
            #endif
            
            // fog
            #pragma multi_compile_fog 

            struct VertexInput
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };

            struct VertexOutput
            {
               float4 vertex : SV_POSITION;
               float2 uv : TEXCOORD0;
               float fogCoord : TEXCOORD1;
               float3 normal : NORMAL;

               // float3 viewDir : TEXCOORD2;

               // built in view vector


               float3 WorldSpaceViewDirection : TEXCOORD3;
            };
             
            CBUFFER_START(UnityPerMaterial)

              half4 _TintColor;
              half _Alpha;

              #if _SAMPLE_LC | _SAMPLE_LW2
                sampler2D _MainTex;
                float4 _MainTex_ST;

             #elif _SAMPLE_LW1
                 TEXTURE2D(_MainTex);
                 SAMPLER(sampler_MainTex);
                 float4 _Offset;
              
              #endif     

               
            CBUFFER_END
                                  

            VertexOutput vert(VertexInput v)

            {
                VertexOutput o;       
    
                o.normal = TransformObjectToWorld(v.normal);
                // o.normal = normalize(mul(v.normal,(float3x3)UNITY_MATRIX_I_M));               

                o.vertex = TransformWorldToHClip(TransformObjectToWorld(v.vertex.xyz));

                #if _SAMPLE_LC                
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);

                #elif _SAMPLE_LW1 | _SAMPLE_LW2     
                o.uv = v.uv;

                #endif

                 o.fogCoord = ComputeFogFactor(o.vertex.z);
                 o.WorldSpaceViewDirection = _WorldSpaceCameraPos.xyz - mul(GetObjectToWorldMatrix(), float4(v.vertex.xyz, 1.0)).xyz;

               return o;

            }

            half4 frag(VertexOutput i) : SV_Target
            {          
                 
              #if _SAMPLE_LC 
    
              half4 col = tex2D(_MainTex, i.uv);
                  
              #elif _SAMPLE_LW1

              float2 uv = i.uv * _Offset.xy + _Offset.zw;              
              float4 col = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv);    
    
              #elif _SAMPLE_LW2
              float2 uv = i.uv.xy * (_MainTex_ST.xy + _MainTex_ST.zw);
              half4 col = tex2D(_MainTex, uv);
               
              #endif

               #if _LI_LAMBERT
               Light mainLight = GetMainLight();

               float NdotL = saturate(dot(mainLight.direction.xyz, i.normal));
               
               half3 ambient = SampleSH(i.normal);  
               
               col.rgb *= NdotL * mainLight.color.rgb + ambient;

               #elif _LI_BLINNPHONG

               Light mainLight = GetMainLight();

               half3 BlinnPhong = LightingSpecular(mainLight.color.rgb, mainLight.direction.xyz, i.normal, i.WorldSpaceViewDirection.xyz, half4(1,1,0,0), 12);

               half3 ambient = SampleSH(i.normal);  

               col.rgb *= BlinnPhong + ambient;

               #endif

              col *= _TintColor;
              col.rgb = MixFog(col.rgb, i.fogCoord);

              //alphaTest


              //clip(col.a - _Alpha);


              return col;              
            }

            ENDHLSL
    
        }
    }
}
