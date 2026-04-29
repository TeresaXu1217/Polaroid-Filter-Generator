import React, { useState, useEffect, useRef } from 'react';
import { Upload, Download, Image as ImageIcon, Type, Sliders, Check } from 'lucide-react';

// 解析與產生 Lightroom Tone Curve 的色彩查找表 (Look-Up Table)
const generateLUT = (points) => {
  const lut = new Uint8Array(256);
  let pIdx = 0;
  for (let i = 0; i < 256; i++) {
    // 當前亮度值超過目前區段時，移動到下一個控制點區段
    while (pIdx < points.length - 2 && i > points[pIdx + 1][0]) {
      pIdx++;
    }
    const p1 = points[pIdx];
    const p2 = points[pIdx + 1];
    const t = p2[0] === p1[0] ? 0 : (i - p1[0]) / (p2[0] - p1[0]);
    const val = p1[1] + t * (p2[1] - p1[1]);
    lut[i] = Math.min(255, Math.max(0, Math.round(val)));
  }
  return lut;
};

// 來自 Vintage (1).lrtemplate 的 RGB 通道曲線控制點
const rPoints = [[0, 0], [36, 42], [72, 66], [109, 85], [145, 113], [182, 148], [218, 182], [255, 215]];
const gPoints = [[0, 15], [36, 40], [72, 71], [109, 104], [145, 132], [182, 160], [218, 186], [255, 211]];
const bPoints = [[0, 12], [36, 52], [72, 75], [109, 94], [145, 114], [182, 141], [218, 169], [255, 185]];

const rLUT = generateLUT(rPoints);
const gLUT = generateLUT(gPoints);
const bLUT = generateLUT(bPoints);

export default function App() {
  const canvasRef = useRef(null);
  const [imageObj, setImageObj] = useState(null);
  const [title, setTitle] = useState('My Summer Memories');
  const [text1, setText1] = useState('Kyoto, Japan');
  const [text2, setText2] = useState('2023.08.15');
  const [applyFilter, setApplyFilter] = useState(true);

  // 處理照片上傳
  const handleImageUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (event) => {
        const img = new Image();
        img.onload = () => setImageObj(img);
        img.src = event.target.result;
      };
      reader.readAsDataURL(file);
    }
  };

  // 重新繪製 Canvas
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');

    // 設定拍立得輸出的解析度大小 (比例 4:5)
    const cw = 1080;
    const ch = 1350;
    canvas.width = cw;
    canvas.height = ch;

    // 1. 繪製拍立得白色相紙背景
    ctx.fillStyle = '#fafafa'; 
    ctx.fillRect(0, 0, cw, ch);

    const margin = 60;
    const imgSize = cw - margin * 2; // 960

    if (imageObj) {
      // 建立離線 Canvas 來處理圖片裁切與濾鏡
      const tempCanvas = document.createElement('canvas');
      tempCanvas.width = imgSize;
      tempCanvas.height = imgSize;
      const tCtx = tempCanvas.getContext('2d');

      // 裁切成正方形並縮放 (Cover mode)
      const scale = Math.max(imgSize / imageObj.width, imgSize / imageObj.height);
      const w = imageObj.width * scale;
      const h = imageObj.height * scale;
      const x = (imgSize - w) / 2;
      const y = (imgSize - h) / 2;

      tCtx.drawImage(imageObj, x, y, w, h);

      // 如果有勾選，套用色彩濾鏡
      if (applyFilter) {
        const imgData = tCtx.getImageData(0, 0, imgSize, imgSize);
        const data = imgData.data;
        for (let i = 0; i < data.length; i += 4) {
          data[i] = rLUT[data[i]];         // 處理 Red
          data[i + 1] = gLUT[data[i + 1]]; // 處理 Green
          data[i + 2] = bLUT[data[i + 2]]; // 處理 Blue
          // Alpha 通道保持不變
        }
        tCtx.putImageData(imgData, 0, 0);
      }

      // 將處理好的圖片繪製到主 Canvas 上
      ctx.drawImage(tempCanvas, margin, margin);

      // 繪製照片內陰影/邊框以增加立體感
      ctx.strokeStyle = 'rgba(0,0,0,0.05)';
      ctx.lineWidth = 4;
      ctx.strokeRect(margin, margin, imgSize, imgSize);
    } else {
      // 如果未上傳圖片，繪製佔位區塊
      ctx.fillStyle = '#e5e7eb';
      ctx.fillRect(margin, margin, imgSize, imgSize);
      ctx.fillStyle = '#9ca3af';
      ctx.font = '40px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('請從左側上傳照片', cw / 2, margin + imgSize / 2);
    }

    // 2. 繪製文字區域
    ctx.textAlign = 'center';
    const fontStack = '"Noto Sans TC", "Microsoft JhengHei", "PingFang TC", sans-serif';

    // 主標題
    ctx.fillStyle = '#1f2937';
    ctx.font = `bold 54px ${fontStack}`;
    ctx.fillText(title || ' ', cw / 2, 1120);

    // 內文第一行
    ctx.font = `40px ${fontStack}`;
    ctx.fillStyle = '#4b5563';
    ctx.fillText(text1 || ' ', cw / 2, 1200);

    // 內文第二行
    ctx.font = `italic 32px ${fontStack}`;
    ctx.fillStyle = '#6b7280';
    ctx.fillText(text2 || ' ', cw / 2, 1260);

  }, [imageObj, title, text1, text2, applyFilter]);

  // 處理下載
  const handleDownload = () => {
    if (!canvasRef.current) return;
    const dataUrl = canvasRef.current.toDataURL('image/jpeg', 0.95);
    const link = document.createElement('a');
    link.download = `polaroid-${Date.now()}.jpg`;
    link.href = dataUrl;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  return (
    <div className="min-h-screen bg-gray-100 py-10 px-4 font-sans">
      <div className="max-w-6xl mx-auto">
        <header className="mb-10 text-center">
          <h1 className="text-3xl font-bold text-gray-800 flex items-center justify-center gap-3">
            <ImageIcon className="w-8 h-8 text-indigo-600" />
            拍立得復古濾鏡產生器
          </h1>
          <p className="text-gray-500 mt-2">將照片套用你專屬的 Lightroom 復古色彩，並添加文字與相紙邊框。</p>
        </header>

        <div className="flex flex-col lg:flex-row gap-8 bg-white p-8 rounded-2xl shadow-sm border border-gray-200">
          {/* 左側：控制面板 */}
          <div className="w-full lg:w-1/2 space-y-6">
            
            {/* 照片上傳 */}
            <div className="space-y-2">
              <label className="block text-sm font-semibold text-gray-700 flex items-center gap-2">
                <Upload className="w-4 h-4" /> 步驟一：上傳照片
              </label>
              <label className="flex justify-center w-full h-32 px-4 transition bg-white border-2 border-gray-300 border-dashed rounded-xl appearance-none cursor-pointer hover:border-indigo-400 hover:bg-indigo-50 focus:outline-none">
                <span className="flex items-center space-x-2">
                  <svg xmlns="http://www.w3.org/2000/svg" className="w-6 h-6 text-gray-600" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth="2">
                    <path strokeLinecap="round" strokeLinejoin="round" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
                  </svg>
                  <span className="font-medium text-gray-600">
                    {imageObj ? '點擊重新選擇照片' : '點擊選擇檔案上傳'}
                  </span>
                </span>
                <input type="file" accept="image/*" name="file_upload" className="hidden" onChange={handleImageUpload} />
              </label>
            </div>

            {/* 文字輸入區域 */}
            <div className="space-y-4">
              <label className="block text-sm font-semibold text-gray-700 flex items-center gap-2">
                <Type className="w-4 h-4" /> 步驟二：編輯文字
              </label>
              
              <div>
                <label className="block text-xs text-gray-500 mb-1">主標題</label>
                <input 
                  type="text" 
                  value={title} 
                  onChange={(e) => setTitle(e.target.value)}
                  className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                  placeholder="輸入標題文字..."
                />
              </div>

              <div>
                <label className="block text-xs text-gray-500 mb-1">內文一 (如地點)</label>
                <input 
                  type="text" 
                  value={text1} 
                  onChange={(e) => setText1(e.target.value)}
                  className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                  placeholder="輸入地點或描述..."
                />
              </div>

              <div>
                <label className="block text-xs text-gray-500 mb-1">內文二 (如日期)</label>
                <input 
                  type="text" 
                  value={text2} 
                  onChange={(e) => setText2(e.target.value)}
                  className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:outline-none"
                  placeholder="輸入日期..."
                />
              </div>
            </div>

            {/* 濾鏡開關 */}
            <div className="pt-2">
              <label className="block text-sm font-semibold text-gray-700 flex items-center gap-2 mb-3">
                <Sliders className="w-4 h-4" /> 步驟三：復古濾鏡
              </label>
              <button
                onClick={() => setApplyFilter(!applyFilter)}
                className={`w-full flex items-center justify-between px-4 py-3 rounded-xl border transition-colors ${applyFilter ? 'bg-indigo-50 border-indigo-200 text-indigo-700' : 'bg-gray-50 border-gray-200 text-gray-600'}`}
              >
                <span className="font-medium">套用 Polaroid •••• 色彩檔案</span>
                <div className={`w-6 h-6 rounded-md flex items-center justify-center ${applyFilter ? 'bg-indigo-600 text-white' : 'bg-gray-200'}`}>
                  {applyFilter && <Check className="w-4 h-4" />}
                </div>
              </button>
            </div>

            {/* 下載按鈕 */}
            <div className="pt-4">
               <button 
                  onClick={handleDownload}
                  disabled={!imageObj}
                  className="w-full flex items-center justify-center gap-2 bg-gray-900 hover:bg-gray-800 disabled:bg-gray-300 disabled:cursor-not-allowed text-white py-4 px-6 rounded-xl font-bold transition shadow-lg shadow-gray-200"
               >
                  <Download className="w-5 h-5" />
                  匯出並下載圖片
               </button>
            </div>
          </div>

          {/* 右側：Canvas 預覽 */}
          <div className="w-full lg:w-1/2 flex items-center justify-center bg-gray-50 rounded-xl border border-gray-200 p-6">
            <div className="relative shadow-2xl overflow-hidden rounded-md" style={{ maxWidth: '400px', width: '100%' }}>
              <canvas 
                ref={canvasRef} 
                className="w-full h-auto block"
              />
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
