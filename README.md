通常二维码由服务器生成，以图片格式发送到客户端，由客户端直接展示。

二维码也可以由客户端使用javascript或其他内置的SDK直接生成。

#### 0、需求
二维码生成过程中往往是先生成矩阵，然后使用矩阵生成图片，矩阵就是由01组成的一维或二位数组。

例如，由`ZXing`生成的`ByteMatrix`就是一个由行列数据组成的二维数组。
```csharp
//可以生成由01组成的一个矩阵字符串。
private string GetMatrixString(ByteMatrix matrix)
{
    return string.Join("", matrix.Array.Select(t => string.Join("", t)));
}
```
有时候，我们需要尽可能的减少网络传输，对二维码进行缓存处理，或者减少二维码矩阵生成的逻辑。

这时，我们完全可以将这个字符串发送给客户端，再由客户端生成图片，减少网络浏览传输或者方便客户端缓存二维码。

下面方法可以对矩阵处理，生成二维码图片。
```javascript
function createQRCodeCanvas(matrix, size, padding) {
  size = size || 3;
  padding = padding === undefined ? 3 : padding;
  const width = Math.sqrt(matrix.length);
  const canvasWith = width * size + padding * 2;

  const canvas = document.createElement('canvas');
  canvas.width = canvasWith;
  canvas.height = canvasWith;

  const ctx = canvas.getContext('2d');
  ctx.fillStyle = "rgb(0,0,0)";


  for (let y = 0; y < width; y++) {
    for (let x = 0; x < width; x++) {
      const point = y * width + x;
      if (matrix[point] === 1) {
        ctx.fillRect(padding + x * size, padding + y * size, size, size);
      }
    }
  }
  return canvas.toDataURL();
}
```
#### 1、矩阵压缩
由于矩阵完全由01组成，我们可以对矩阵进行处理，每8位作为一组，转换成一个字节。

往往矩阵的长度不会被8整除，所以我们在最后一位补1，标识矩阵结束，哪怕矩阵长度能被8整除，我们也补1。

下面代码生成压缩后的矩阵字节数组。
```csharp
private byte[] GetMatrixBytes(ByteMatrix matrix)
{
    var qrData = matrix.Array;
    int idx = 7;
    int count = 0;
    byte[] result = new byte[(int)Math.Ceiling((decimal)(qrData.Length * qrData.Length + 1) / 8)];

    for (int i = 0; i < qrData.Length; i++)
    {
        byte[] line = qrData[i];
        for (int j = 0; j < line.Length; j++)
        {
            result[count++ >> 3] |= (byte)(line[j] << idx--);
            if (idx == -1) idx = 7;
        }
    }
    result[count >> 3] |= (byte)(1 << idx); //最后一位补1
    return result;
}
```
生成矩阵字节数组后，可以转换成base64发送到客户端，这样会大大减少传输的数据量。

#### 2、矩阵还原
将上面的算法逆转即可。

例如，用csharp还原。
```csharp
/// <summary>
/// 从字节数组还原矩阵字符串
/// </summary>
/// <param name="matrix"></param>
/// <returns></returns>
private byte[] GetMatrixBytes(byte[] matrix)
{

    byte[] bytes = new byte[matrix.Length * 8];

    int idx = 0;

    foreach (byte chr in matrix) for (int i = 7; i >= 0; i--) bytes[idx++] = (byte)((chr >> i) & 1);

    while (bytes[--idx] == 0) ;

    return bytes.Take(idx).ToArray();
}
```
用javascript还原
```javascript
function getMatrix(raws) {

  const bytes = [];

  let idx = 0;

  for (let j = 0; j < raws.length; j++) {
    for (let i = 7; i >= 0; i--) bytes[idx++] = (raws[j] >> i) & 1;
  }

  while (bytes[--idx] === 0) ;

  return bytes.slice(0, idx);
}
```
矩阵还原出来后，就可以用文章最开始的方法将矩阵生成图片了。

### 3、总结
通过对矩阵的处理，进一步减少标识矩阵所用的字节数，从而减少网络传输的数据，并且更方便的缓存生成的二维码。

客户端可以只缓存压缩后的矩阵，必要的时候还原并展示即可。
