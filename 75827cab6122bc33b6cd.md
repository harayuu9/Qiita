<!--
title:   [Unity] ILPostProcessorを用いてHash値のコンパイル時計算を試みた
tags:    C#,IL,ILPostProcessor,Unity
id:      75827cab6122bc33b6cd
private: false
-->
C++のconstexprに憧れて何か代用出来ないだろうかと考えていた時に
VContainerがILPostProcessorで最適化してたのを思い出して同じ手法を試してみようと思いました。

リポジトリ

https://github.com/harayuu9/ILPostProcessorTest

# はじめに
C#でconstexprの発想は@pCYSl5EDgo様の記事(
https://qiita.com/pCYSl5EDgo/items/5846ce9255bf81b37807 )でなるほどと思っていました、感謝します。

# ILPostProcessor
  * ILPostProcessorとはUnityが提供しているコンパイル時に処理を差し込める機能でまだ標準化はされてないです
  * コンパイル中のAssemblyDefinitionの書き換えを行うことが出来ます
 
# 目標
## CSharp
目標として以下のコードが変換されれば良いでしょう

```cs
var hash = Hash.Runtime.Hash.CalcHash("Hello World");
// hash = -15678624
```

`cs
var hash = -15678624
`

## IL
ILの変換をしないといけないのでCSharpの目標からILの目標に考え直してみます

```
ldstr  Hello World
call  System.Int32 Hash.Runtime.Hash::CalcHash(System.String)
```

`
ldc.i4  -1884438777
`

# ICompiledAssembly
AssemblyDefinitionを取得するにはILPostProcessorの引数になっているICompiledAssemblyから変換しないといけません
単純には行かないようでECS,MLAPI,VContainerを見てみましたがどれもほぼECSのコピペだったのでそういうものだと思って使います
なので変換手順については解説 ~~出来ません~~しません

# 変換
Hashを参照しているAssemblyDefinitionの関数全てを取得してILの解析→上記のILコードがあり次第変換していきます。

```cs
    public override ILPostProcessResult Process(ICompiledAssembly compiledAssembly)
    {
        if (!WillProcess(compiledAssembly))
            return null;

        var assemblyDefinition = Utils.LoadAssemblyDefinition(compiledAssembly);

        var hashMap = new Dictionary<string, int>();

        var builder = new StringBuilder();
        
        void TryGenerateType(TypeDefinition typeDef)
        {
            foreach (var method in typeDef.Methods)
            {
                var processor = method.Body.GetILProcessor();

                for (var index = 0; index < processor.Body.Instructions.Count; index++)
                {
                    var bodyInstruction = processor.Body.Instructions[index];
                    if (bodyInstruction.OpCode == OpCodes.Call && bodyInstruction.Previous.OpCode == OpCodes.Ldstr &&
                        bodyInstruction.Operand.ToString() == "System.Int32 Hash.Runtime.Hash::CalcHash(System.String)")
                    {
                        var hashStr = bodyInstruction.Previous.Operand.ToString();
                        int hash;
                        if (hashMap.ContainsKey(hashStr))
                        {
                            hash = hashMap[hashStr];
                        }
                        else
                        {
                            hash = Animator.StringToHash(hashStr);
                            hashMap.Add(hashStr, hash);
                        }

                        var remove1 = processor.Body.Instructions[index - 1];
                        var remove2 = processor.Body.Instructions[index];
                        var ins     = processor.Create(OpCodes.Ldc_I4, hash);
                        foreach (var instruction in processor.Body.Instructions)
                        {
                            if (instruction.Previous == remove2)
                                instruction.Previous = ins;
                            if (instruction.Next == remove1)
                                instruction.Next = ins;
                            if (instruction.Operand is Instruction)
                            {
                                if (instruction.Operand == remove1 || instruction.Operand == remove2)
                                    instruction.Operand = ins;
                            }
                        }
                        
                        processor.Body.Instructions.RemoveAt(index - 1);
                        processor.Body.Instructions.RemoveAt(index - 1);
                        processor.Body.Instructions.Insert(index - 1, ins);

                        builder.AppendLine(method.Name);
                        foreach (var instruction in processor.Body.Instructions)
                        {
                            builder.AppendLine(instruction + "  " + instruction.Operand?.GetType());
                        }

                        builder.AppendLine();
                    }
                }
            }
        }

        foreach (var typeDef in assemblyDefinition.MainModule.Types.Where(typeDef => typeDef.FullName != "<Module>"))
        {
            TryGenerateType(typeDef);
        }

        foreach (var keyValuePair in hashMap)
        {
            builder.AppendLine(keyValuePair.Key + "  " + keyValuePair.Value);
        }
        File.WriteAllText(logSavePath, builder.ToString());
        
        var pe  = new MemoryStream();
        var pdb = new MemoryStream();

        var writeParameter = new WriterParameters
        {
            SymbolWriterProvider = new PortablePdbWriterProvider(),
            SymbolStream         = pdb,
            WriteSymbols         = true
        };

        assemblyDefinition.Write(pe, writeParameter);

        return new ILPostProcessResult(new InMemoryAssembly(pe.ToArray(), pdb.ToArray()), null);
    }
```

単純に特定の関数を使っているところを全検索して置き換えれるかどうか調べています。
置き換えれる物に関しては上記のようにIL命令2つを取り除いて新しい命令1つを挿入しています。
また別の箇所でJump命令等で参照されているので解決をしています。(前後は置き換え不要かも
Hashなのでこの段階で衝突チェックをしておくと実務では優しいと思います。

# 検証
本当に高速に動くか検証してみましょう

```cs
public class Simple : MonoBehaviour
{
    private void Start()
    {
        var sw = new Stopwatch();
        sw.Start();

        for (var i = 0; i < 10000000; i++)
        {
            Hash.Runtime.Hash.CalcHash("少し長い単語のハッシュ値を計算して本当に早いかどうか確かめたいと思っているのですがいかがでしょうか10");
        }
        sw.Stop();
        Debug.Log("constexpr hash:" + sw.ElapsedMilliseconds + " hash:" + Hash.Runtime.Hash.CalcHash("少し長い単語のハッシュ値を計算して本当に早いかどうか確かめたいと思っているのですがいかがでしょうか10"));

        var       str = "少し長い単語のハッシュ値を計算して本当に早いかどうか確かめたいと思っているのですがいかがでしょうか";
        const int cnt = 10;
        str += cnt;
        sw.Restart();
        for (var i = 0; i < 10000000; i++)
        {
            Hash.Runtime.Hash.CalcHash(str);
        }
        sw.Stop();
        Debug.Log("default hash:" + sw.ElapsedMilliseconds + " hash:" + Hash.Runtime.Hash.CalcHash(str));
    }
}
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/7f8c3b55-519e-838f-216d-deb997a81efa.png)
しっかりと動いててよかったです。ILも一応確認してみます(少し長いです)

```
IL_0000: newobj System.Void System.Diagnostics.Stopwatch::.ctor()  Mono.Cecil.MethodReference
IL_0005: stloc.0  
IL_0006: ldloc.0  
IL_0007: callvirt System.Void System.Diagnostics.Stopwatch::Start()  Mono.Cecil.MethodReference
IL_000c: ldc.i4.0  
IL_000d: stloc.2  
IL_000e: br.s IL_001f  Mono.Cecil.Cil.Instruction
IL_0000: ldc.i4 -1245539690  System.Int32
IL_001a: pop  
IL_001b: ldloc.2  
IL_001c: ldc.i4.1  
IL_001d: add  
IL_001e: stloc.2  
IL_001f: ldloc.2  
IL_0020: ldc.i4 10000000  System.Int32
IL_0025: blt.s IL_0000  Mono.Cecil.Cil.Instruction
IL_0027: ldloc.0  
IL_0028: callvirt System.Void System.Diagnostics.Stopwatch::Stop()  Mono.Cecil.MethodReference
IL_002d: ldc.i4.4  
IL_002e: newarr System.Object  Mono.Cecil.TypeReference
IL_0033: dup  
IL_0034: ldc.i4.0  
IL_0035: ldstr "constexpr hash:"  System.String
IL_003a: stelem.ref  
IL_003b: dup  
IL_003c: ldc.i4.1  
IL_003d: ldloc.0  
IL_003e: callvirt System.Int64 System.Diagnostics.Stopwatch::get_ElapsedMilliseconds()  Mono.Cecil.MethodReference
IL_0043: box System.Int64  Mono.Cecil.TypeReference
IL_0048: stelem.ref  
IL_0049: dup  
IL_004a: ldc.i4.2  
IL_004b: ldstr " hash:"  System.String
IL_0050: stelem.ref  
IL_0051: dup  
IL_0052: ldc.i4.3  
IL_0000: ldc.i4 -1245539690  System.Int32
IL_005d: box System.Int32  Mono.Cecil.TypeReference
IL_0062: stelem.ref  
IL_0063: call System.String System.String::Concat(System.Object[])  Mono.Cecil.MethodReference
IL_0068: call System.Void UnityEngine.Debug::Log(System.Object)  Mono.Cecil.MethodReference
IL_006d: ldstr "少し長い単語のハッシュ値を計算して本当に早いかどうか確かめたいと思っているのですがいかがでしょうか"  System.String
IL_0072: stloc.1  
IL_0073: ldloc.1  
IL_0074: ldc.i4.s 10  System.SByte
IL_0076: box System.Int32  Mono.Cecil.TypeReference
IL_007b: call System.String System.String::Concat(System.Object,System.Object)  Mono.Cecil.MethodReference
IL_0080: stloc.1  
IL_0081: ldloc.0  
IL_0082: callvirt System.Void System.Diagnostics.Stopwatch::Restart()  Mono.Cecil.MethodReference
IL_0087: ldc.i4.0  
IL_0088: stloc.3  
IL_0089: br.s IL_0096  Mono.Cecil.Cil.Instruction
IL_008b: ldloc.1  
IL_008c: call System.Int32 Hash.Runtime.Hash::CalcHash(System.String)  Mono.Cecil.MethodReference
IL_0091: pop  
IL_0092: ldloc.3  
IL_0093: ldc.i4.1  
IL_0094: add  
IL_0095: stloc.3  
IL_0096: ldloc.3  
IL_0097: ldc.i4 10000000  System.Int32
IL_009c: blt.s IL_008b  Mono.Cecil.Cil.Instruction
IL_009e: ldloc.0  
IL_009f: callvirt System.Void System.Diagnostics.Stopwatch::Stop()  Mono.Cecil.MethodReference
IL_00a4: ldc.i4.4  
IL_00a5: newarr System.Object  Mono.Cecil.TypeReference
IL_00aa: dup  
IL_00ab: ldc.i4.0  
IL_00ac: ldstr "default hash:"  System.String
IL_00b1: stelem.ref  
IL_00b2: dup  
IL_00b3: ldc.i4.1  
IL_00b4: ldloc.0  
IL_00b5: callvirt System.Int64 System.Diagnostics.Stopwatch::get_ElapsedMilliseconds()  Mono.Cecil.MethodReference
IL_00ba: box System.Int64  Mono.Cecil.TypeReference
IL_00bf: stelem.ref  
IL_00c0: dup  
IL_00c1: ldc.i4.2  
IL_00c2: ldstr " hash:"  System.String
IL_00c7: stelem.ref  
IL_00c8: dup  
IL_00c9: ldc.i4.3  
IL_00ca: ldloc.1  
IL_00cb: call System.Int32 Hash.Runtime.Hash::CalcHash(System.String)  Mono.Cecil.MethodReference
IL_00d0: box System.Int32  Mono.Cecil.TypeReference
IL_00d5: stelem.ref  
IL_00d6: call System.String System.String::Concat(System.Object[])  Mono.Cecil.MethodReference
IL_00db: call System.Void UnityEngine.Debug::Log(System.Object)  Mono.Cecil.MethodReference
IL_00e0: ret   
```

しっかり置き換わってますね。

# 最後に

気軽にコンパイル時に処理を仕込めると様々な黒魔術ができそうですね。
ILPostProcessorの解説記事や最小構成のサンプルがなかったので書いてみました。
ILだけ触りたいのに他の部分調べるの面倒な人は参考にしてみてください 

* 完全なconstexprの作成
* 書く度に増えるカウンター変数
* ラムダ式のアロケーションを撲滅
* ゼロアロケーションなLinqライブラリ

ちなみにですが目的は高速化だけではなくHashにする前の文字列リテラルをソースコードに残らないというのも業務では嬉しいことではないでしょうか？　
UnityだとIL2CPPしてないものは簡単に見られてしまいますし、IL2CPPしても文字列リテラルは消えないので頑張れば見えてしまします。

https://github.com/harayuu9/ILPostProcessorTest