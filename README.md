import React, { useState, useMemo, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import {
  getAuth,
  signInAnonymously,
  signInWithCustomToken,
  onAuthStateChanged,
} from 'firebase/auth';
import {
  getFirestore,
  doc,
  setDoc,
  onSnapshot,
} from 'firebase/firestore';
import {
  Calendar,
  ShoppingCart,
  BookOpen,
  Plus,
  Trash2,
  Sparkles,
  Sliders,
  Users,
  Check,
  Copy,
  Coffee,
  Utensils,
  Sun,
  Moon,
  ChefHat,
  Video,
  Edit3,
  Heart,
  X,
  RefreshCw,
  ChevronRight,
  Info,
  Minus,
  Activity,
  AlertTriangle,
  CheckCircle2,
  Sparkle,
  Cloud
} from 'lucide-react';

// クラウド自動保存サービスの初期化
const firebaseConfig =
  typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
const firebaseApp = firebaseConfig ? initializeApp(firebaseConfig) : null;
const auth = firebaseApp ? getAuth(firebaseApp) : null;
const db = firebaseApp ? getFirestore(firebaseApp) : null;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// ============================================================================
// 定数・初期データ定義
// ============================================================================

// 日本人の食事摂取基準（3〜5歳児の1日推奨量・目安量）
const CHILD_DAILY_NUTRITION_TARGETS = {
  calories: { label: 'エネルギー', unit: 'kcal', daily: 1250, isUpperLimit: false },
  protein: { label: 'タンパク質', unit: 'g', daily: 25, isUpperLimit: false },
  calcium: { label: 'カルシウム', unit: 'mg', daily: 550, isUpperLimit: false },
  iron: { label: '鉄分', unit: 'mg', daily: 5.0, isUpperLimit: false },
  fiber: { label: '食物繊維', unit: 'g', daily: 8.0, isUpperLimit: false },
  salt: { label: '食塩相当量', unit: 'g', daily: 3.5, isUpperLimit: true }, // 上限目標
};

// 3歳・4歳児の基本量（大人100%に対する基準比率 = 75%）
const CHILD_BASE_ADULT_RATIO = 0.75;

// 一緒に食べる主食（炭水化物）の選択肢（3〜4歳基準1.0倍あたりの1食分栄養価）
const STAPLE_OPTIONS = [
  {
    id: 'none',
    label: '主食なし（単品）',
    childPortionLabel: 'なし',
    nutrition: { calories: 0, protein: 0, calcium: 0, iron: 0, fiber: 0, salt: 0 },
  },
  {
    id: 'rice-white',
    label: '白ごはん',
    childPortionLabel: '子供茶碗1杯 (110g)',
    nutrition: { calories: 172, protein: 2.8, calcium: 3, iron: 0.1, fiber: 1.6, salt: 0 },
  },
  {
    id: 'rice-mixed',
    label: '雑穀・もち麦ごはん',
    childPortionLabel: '子供茶碗1杯 (110g)',
    nutrition: { calories: 170, protein: 3.3, calcium: 12, iron: 0.5, fiber: 2.8, salt: 0 },
  },
  {
    id: 'bread',
    label: '食パン（6枚切り）',
    childPortionLabel: '0.75枚 (45g)',
    nutrition: { calories: 112, protein: 4.0, calcium: 13, iron: 0.3, fiber: 1.9, salt: 0.5 },
  },
  {
    id: 'udon',
    label: 'うどん（麺のみ）',
    childPortionLabel: '0.75玉 (150g)',
    nutrition: { calories: 143, protein: 3.9, calcium: 8, iron: 0.3, fiber: 1.2, salt: 0.3 },
  },
];

// わが家の定番「ハーバードスープ（4種野菜の重ね煮スープ）」の子供1杯(150ml)あたり栄養価
const HARVARD_SOUP_OPTION = {
  label: 'わが家のハーバードスープ',
  childPortionLabel: '子供お椀1杯 (野菜たっぷり)',
  nutrition: {
    calories: 48,
    protein: 1.4,
    calcium: 26,
    iron: 0.5,
    fiber: 2.9, // キャベツ・玉ねぎ・にんじん・かぼちゃ由来の豊富な食物繊維
    salt: 0.0, // 基本は塩なし（素材の甘み）
  },
};

// 一緒によく食べる「Q・B・Bチーズ（5個入りパックの1個）」あたり栄養価
const QBB_CHEESE_OPTION = {
  label: 'Q・B・Bチーズ（5個入りパックの1個）',
  childPortionLabel: '5個入りパックの1個',
  nutrition: {
    calories: 47,
    protein: 3.2,
    calcium: 95, // カルシウム・タンパク質補給に最適
    iron: 0.05,
    fiber: 0.0,
    salt: 0.4,
  },
};

// デザート（果物）の選択肢（3〜4歳児の1食目安量あたり栄養価）
const DESSERT_FRUIT_OPTIONS = [
  {
    id: 'none',
    label: 'デザートなし',
    childPortionLabel: 'なし',
    nutrition: { calories: 0, protein: 0, calcium: 0, iron: 0, fiber: 0, salt: 0 },
  },
  {
    id: 'apple',
    label: 'リンゴ',
    childPortionLabel: '1/4個 (約60g)',
    nutrition: { calories: 34, protein: 0.1, calcium: 2, iron: 0.1, fiber: 0.9, salt: 0 },
  },
  {
    id: 'mandarin',
    label: 'ミカン',
    childPortionLabel: '1個 (約75g)',
    nutrition: { calories: 37, protein: 0.5, calcium: 16, iron: 0.1, fiber: 0.8, salt: 0 },
  },
  {
    id: 'kiwi',
    label: 'キウイ',
    childPortionLabel: '1/2個 (約50g)',
    nutrition: { calories: 26, protein: 0.5, calcium: 13, iron: 0.1, fiber: 1.3, salt: 0 },
  },
  {
    id: 'persimmon',
    label: '柿',
    childPortionLabel: '1/4個 (約50g)',
    nutrition: { calories: 32, protein: 0.2, calcium: 5, iron: 0.1, fiber: 0.8, salt: 0 },
  },
  {
    id: 'banana',
    label: 'バナナ',
    childPortionLabel: '1/2本 (約45g)',
    nutrition: { calories: 42, protein: 0.5, calcium: 3, iron: 0.1, fiber: 0.5, salt: 0 },
  },
];

const DAYS = [
  { key: 'mon', label: '月', fullLabel: '月曜日' },
  { key: 'tue', label: '火', fullLabel: '火曜日' },
  { key: 'wed', label: '水', fullLabel: '水曜日' },
  { key: 'thu', label: '木', fullLabel: '木曜日' },
  { key: 'fri', label: '金', fullLabel: '金曜日' },
  { key: 'sat', label: '土', fullLabel: '土曜日' },
  { key: 'sun', label: '日', fullLabel: '日曜日' },
];

const MEAL_TYPES = [
  { key: 'breakfast', label: '朝', icon: Sun, color: 'text-amber-500', bg: 'bg-amber-50' },
  { key: 'lunch', label: '昼', icon: Utensils, color: 'text-emerald-600', bg: 'bg-emerald-50' },
  { key: 'dinner', label: '晩', icon: Moon, color: 'text-indigo-600', bg: 'bg-indigo-50' },
];

const INGREDIENT_CATEGORIES = [
  '野菜・果物',
  '肉・魚・卵',
  '冷蔵・乳製品・パン',
  '調味料・油',
  '乾物・米・その他',
];

// 初期レシピ＆定番メニューデータ（開いてすぐ使えるサンプル）
const INITIAL_RECIPES = [
  {
    id: 'preset-breakfast-bread',
    title: '定番の朝パン＆フルーツセット',
    type: 'preset',
    servings: 3.5, // 家族4人（大人2＋子2）の1回分基準
    sourceUrl: '',
    tags: ['朝食定番', '時短'],
    // 大人1人前（100%）あたりの栄養素目安
    nutritionPerServing: {
      calories: 380,
      protein: 13.5,
      calcium: 260,
      iron: 1.1,
      fiber: 3.2,
      salt: 0.9,
    },
    childNutritionAdvice: 'カルシウムとタンパク質がしっかり摂れる朝食です。鉄分が少し少なめなので、きな粉をヨーグルトに混ぜたり、プルーンやレーズンを添えるとさらに栄養バランスが整います。',
    ingredients: [
      { id: 'i1', name: '食パン（6枚切り）', amount: 3, unit: '枚', category: '冷蔵・乳製品・パン', isSeasoning: false, customMultiplier: 1 },
      { id: 'i2', name: 'バナナ', amount: 2, unit: '本', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i3', name: 'プレーンヨーグルト', amount: 250, unit: 'g', category: '冷蔵・乳製品・パン', isSeasoning: false, customMultiplier: 1 },
      { id: 'i4', name: '牛乳', amount: 400, unit: 'ml', category: '冷蔵・乳製品・パン', isSeasoning: false, customMultiplier: 1 },
    ],
  },
  {
    id: 'preset-breakfast-rice',
    title: '定番の和朝食（納豆・卵・おにぎり）',
    type: 'preset',
    servings: 3.5,
    sourceUrl: '',
    tags: ['朝食定番', '和食'],
    nutritionPerServing: {
      calories: 410,
      protein: 18.0,
      calcium: 110,
      iron: 2.6,
      fiber: 3.8,
      salt: 1.4,
    },
    childNutritionAdvice: '納豆と卵でタンパク質・鉄分が優秀です！カルシウムがやや控えめなので、コップ1杯の牛乳や小魚・チーズをプラスすると完璧です。',
    ingredients: [
      { id: 'i5', name: '納豆', amount: 3, unit: 'パック', category: '冷蔵・乳製品・パン', isSeasoning: false, customMultiplier: 1 },
      { id: 'i6', name: '卵', amount: 3, unit: '個', category: '肉・魚・卵', isSeasoning: false, customMultiplier: 1 },
      { id: 'i7', name: 'ミニトマト', amount: 6, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i8', name: '醤油', amount: 1, unit: '小さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
    ],
  },
  {
    id: 'recipe-nikujaga',
    title: '動画レシピ：フライパンで簡単！ほっこり肉じゃが',
    type: 'video',
    servings: 2, // レシピ動画の基準（2人前）→ 家族3.5人前に自動換算される
    sourceUrl: 'https://www.youtube.com/watch?v=example_nikujaga',
    tags: ['動画レシピ', '晩ごはん'],
    nutritionPerServing: {
      calories: 440,
      protein: 19.5,
      calcium: 45,
      iron: 1.8,
      fiber: 4.1,
      salt: 2.4,
    },
    childNutritionAdvice: 'お肉と野菜でエネルギー・タンパク質・食物繊維がバランス良く摂れます。カルシウムと鉄分を補うため、副菜に「小松菜としらすのおひたし」や「冷奴・お味噌汁（わかめ・豆腐）」を合わせるのがおすすめです。',
    ingredients: [
      { id: 'i9', name: '豚こま切れ肉', amount: 200, unit: 'g', category: '肉・魚・卵', isSeasoning: false, customMultiplier: 1 },
      { id: 'i10', name: 'じゃがいも', amount: 3, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i11', name: '玉ねぎ', amount: 1, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i12', name: 'にんじん', amount: 0.5, unit: '本', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i13', name: '醤油', amount: 2.5, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
      { id: 'i14', name: 'みりん', amount: 2, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
      { id: 'i15', name: '砂糖', amount: 1.5, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
      { id: 'i16', name: '和風だしの素', amount: 1, unit: '小さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
    ],
  },
  {
    id: 'recipe-teriyaki',
    title: '動画レシピ：子供も大好き！やわらか鶏の照り焼き',
    type: 'video',
    servings: 2,
    sourceUrl: 'https://www.youtube.com/watch?v=example_teriyaki',
    tags: ['動画レシピ', 'メイン'],
    nutritionPerServing: {
      calories: 490,
      protein: 28.0,
      calcium: 35,
      iron: 1.4,
      fiber: 1.8,
      salt: 2.3,
    },
    childNutritionAdvice: '成長期に必要なタンパク質がたっぷり摂れます！一方でカルシウム・鉄分・食物繊維が不足しやすいため、ブロッコリーやほうれん草の胡麻和え、牛乳・ヨーグルトを一緒に摂ると栄養満点になります。',
    ingredients: [
      { id: 'i17', name: '鶏もも肉', amount: 300, unit: 'g', category: '肉・魚・卵', isSeasoning: false, customMultiplier: 1 },
      { id: 'i18', name: 'キャベツ（付け合わせ）', amount: 0.25, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i19', name: '醤油', amount: 2, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
      { id: 'i20', name: 'みりん', amount: 2, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
      { id: 'i21', name: '酒', amount: 2, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
      { id: 'i22', name: '砂糖', amount: 1, unit: '大さじ', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
    ],
  },
  {
    id: 'recipe-udon',
    title: 'お昼の定番：具だくさん野菜うどん',
    type: 'preset',
    servings: 3.5,
    sourceUrl: '',
    tags: ['昼ごはん', '麺類'],
    nutritionPerServing: {
      calories: 430,
      protein: 17.5,
      calcium: 85,
      iron: 2.1,
      fiber: 3.4,
      salt: 2.8,
    },
    childNutritionAdvice: 'ほうれん草と豚肉で鉄分・ビタミンも摂れる優秀メニューです。めんつゆの塩分が高くなりやすいため、薄味レバーを75〜80%にして汁を残すようにすると安心です。',
    ingredients: [
      { id: 'i23', name: '冷凍うどん', amount: 3, unit: '玉', category: '乾物・米・その他', isSeasoning: false, customMultiplier: 1 },
      { id: 'i24', name: '豚バラ薄切り肉', amount: 150, unit: 'g', category: '肉・魚・卵', isSeasoning: false, customMultiplier: 1 },
      { id: 'i25', name: 'ほうれん草', amount: 0.5, unit: '束', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i26', name: 'めんつゆ（3倍濃縮）', amount: 80, unit: 'ml', category: '調味料・油', isSeasoning: true, customMultiplier: 1 },
    ],
  },
  {
    id: 'preset-harvard-soup',
    title: 'わが家の定番：ハーバードスープ（4種野菜スープ）',
    type: 'preset',
    servings: 3.5,
    sourceUrl: '',
    tags: ['定番スープ', '野菜たっぷり'],
    nutritionPerServing: {
      calories: 64,
      protein: 1.9,
      calcium: 35,
      iron: 0.7,
      fiber: 3.9,
      salt: 0.0,
    },
    childNutritionAdvice:
      'キャベツ・玉ねぎ・にんじん・かぼちゃの甘みだけで作るハーバードスープは、塩分ゼロで食物繊維とビタミンがたっぷり！どんな主菜にも安心してプラスできる最強の副菜です。',
    ingredients: [
      { id: 'i27', name: 'キャベツ', amount: 150, unit: 'g', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i28', name: '玉ねぎ', amount: 1, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i29', name: 'にんじん', amount: 0.5, unit: '本', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
      { id: 'i30', name: 'かぼちゃ', amount: 150, unit: 'g', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
    ],
  },
];

// 1週間の初期献立サンプルデータ
const INITIAL_PLAN = {
  mon: { breakfast: ['preset-breakfast-bread'], lunch: [], dinner: ['recipe-nikujaga'] },
  tue: { breakfast: ['preset-breakfast-bread'], lunch: [], dinner: ['recipe-teriyaki'] },
  wed: { breakfast: ['preset-breakfast-rice'], lunch: [], dinner: [] },
  thu: { breakfast: ['preset-breakfast-bread'], lunch: [], dinner: [] },
  fri: { breakfast: ['preset-breakfast-rice'], lunch: [], dinner: [] },
  sat: { breakfast: [], lunch: ['recipe-udon'], dinner: [] },
  sun: { breakfast: [], lunch: [], dinner: [] },
};

// 換算後の分量を読みやすくフォーマットするヘルパー関数
function formatAmount(value) {
  if (value === null || value === undefined || Number.isNaN(value)) return '0';
  const rounded = Math.round(value * 10) / 10;
  return Number.isInteger(rounded) ? String(rounded) : rounded.toFixed(1);
}

// 指数バックオフ付きGemini API呼び出し（レシピ動画URL・テキスト解析＆子供栄養分析用）
async function extractRecipeWithGemini(inputText) {
  const apiKey = ''; // 実行環境から自動注入
  const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${apiKey}`;

  const systemPrompt = `あなたは日本の家庭料理・小児栄養管理のプロフェッショナル（管理栄養士）です。
ユーザーが入力した「レシピ動画のURL・タイトル」または「レシピの材料テキスト」から、以下を正確に抽出・推計してください：
1. 料理名（title）
2. 元のレシピの基準人数（servings：不明な場合は2）
3. 材料リスト（ingredients）：各材料が調味料・だし・油類（薄味レバーの調整対象）かどうかを isSeasoning (true/false) で厳密に判定してください。
4. 大人1人前（100%量・調味料100%時）あたりの推定栄養素（nutritionPerServing）：
   - calories (kcal)
   - protein (タンパク質 g)
   - calcium (カルシウム mg)
   - iron (鉄分 mg)
   - fiber (食物繊維 g)
   - salt (食塩相当量 g)
5. 3歳〜4歳の子供が食べた場合の栄養評価と、不足しやすい栄養素を補うための手軽なプラス1品アドバイス（childNutritionAdvice：120文字程度で具体的に）

カテゴリは必ず以下の5つのいずれかにしてください:
"野菜・果物", "肉・魚・卵", "冷蔵・乳製品・パン", "調味料・油", "乾物・米・その他"`;

  const payload = {
    contents: [
      {
        parts: [
          {
            text: `以下のレシピ情報から料理名・基準人数・材料リスト・大人1人前あたりの栄養素・子供向け栄養アドバイスを抽出してください:\n\n${inputText}`,
          },
        ],
      },
    ],
    systemInstruction: {
      parts: [{ text: systemPrompt }],
    },
    generationConfig: {
      responseMimeType: 'application/json',
      responseSchema: {
        type: 'OBJECT',
        properties: {
          title: { type: 'STRING', description: '料理名' },
          servings: { type: 'NUMBER', description: '元のレシピが何人前か（不明な場合は2）' },
          nutritionPerServing: {
            type: 'OBJECT',
            description: '元レシピの大人1人前あたりの推定栄養素',
            properties: {
              calories: { type: 'NUMBER' },
              protein: { type: 'NUMBER' },
              calcium: { type: 'NUMBER' },
              iron: { type: 'NUMBER' },
              fiber: { type: 'NUMBER' },
              salt: { type: 'NUMBER' },
            },
            required: ['calories', 'protein', 'calcium', 'iron', 'fiber', 'salt'],
          },
          childNutritionAdvice: {
            type: 'STRING',
            description: '3〜4歳の子供が食べる際の栄養バランス評価と、不足栄養素を補うプラス1品の提案',
          },
          ingredients: {
            type: 'ARRAY',
            items: {
              type: 'OBJECT',
              properties: {
                name: { type: 'STRING', description: '食材名（例：玉ねぎ、醤油）' },
                amount: { type: 'NUMBER', description: '数値の分量（例：0.5、200、2）' },
                unit: { type: 'STRING', description: '単位（例：個、g、大さじ、小さじ、ml、本）' },
                category: { type: 'STRING', description: '売り場カテゴリ' },
                isSeasoning: {
                  type: 'BOOLEAN',
                  description: '醤油・味噌・塩・砂糖・みりん・酒・めんつゆ・ソース・だし等の調味料ならtrue、肉・野菜等の食材ならfalse',
                },
              },
              required: ['name', 'amount', 'unit', 'category', 'isSeasoning'],
            },
          },
        },
        required: ['title', 'servings', 'nutritionPerServing', 'childNutritionAdvice', 'ingredients'],
      },
    },
  };

  let delay = 1000;
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
      if (!response.ok) {
        throw new Error(`HTTP error: ${response.status}`);
      }
      const result = await response.json();
      const text = result?.candidates?.[0]?.content?.parts?.[0]?.text;
      if (!text) throw new Error('Empty response from Gemini');
      return JSON.parse(text);
    } catch (err) {
      if (attempt === 3) throw err;
      await new Promise((res) => setTimeout(res, delay));
      delay *= 2;
    }
  }
}

// ============================================================================
// メインアプリケーションコンポーネント
// ============================================================================

export default function App() {
  // ナビゲーションタブ: 'analyze' (URLで即栄養＆分量チェック) | 'calendar' | 'recipes' | 'shopping'
  const [activeTab, setActiveTab] = useState('analyze');

  // URL栄養分析タブで現在チェック中のレシピIDと基準モード（'meal': 1食目安[1日の1/3] | 'daily': 1日必要量）
  const [selectedAnalyzeRecipeId, setSelectedAnalyzeRecipeId] = useState('recipe-nikujaga');
  const [nutritionBasisMode, setNutritionBasisMode] = useState('meal');
  const [quickUrlInput, setQuickUrlInput] = useState('');
  const [selectedStapleId, setSelectedStapleId] = useState('rice-white'); // 一緒に食べる主食（デフォルト：白ごはん）
  const [includeHarvardSoup, setIncludeHarvardSoup] = useState(true); // ハーバードスープを栄養計算にプラスするか
  const [includeQbbCheese, setIncludeQbbCheese] = useState(false); // キューピーチーズを栄養計算にプラスするか
  const [selectedFruitId, setSelectedFruitId] = useState('none'); // デザート（果物）の選択ID
  const [selectedChildForNutrition, setSelectedChildForNutrition] = useState('jo'); // 'jo' (じょう4歳) | 'so' (そう3歳)

  // 自動保存・同期ステータス管理
  const [user, setUser] = useState(null);
  const [syncStatus, setSyncStatus] = useState('loading'); // 'loading' | 'saving' | 'saved'
  const isRemoteLoadedRef = useRef(false);
  const lastSavedJsonRef = useRef('');

  // 家族4名の食べる量レバー設定（全員「大人100%」を基準に設定）
  // デフォルト: パパ 100%・ママ 100%・じょう(4歳) 75%・そう(3歳) 75%
  const [papaRatio, setPapaRatio] = useState(1.0);
  const [mamaRatio, setMamaRatio] = useState(1.0);
  const [joRatio, setJoRatio] = useState(0.75);
  const [soRatio, setSoRatio] = useState(0.75);

  // 味の濃さ（調味料倍率）調整レバー（デフォルト: 0.80 = 80% 子供向けに少し薄味）
  const [seasoningRatio, setSeasoningRatio] = useState(0.8);

  // レシピ＆定番メニュー一覧
  const [recipes, setRecipes] = useState(INITIAL_RECIPES);

  // 1週間の献立データ { mon: { breakfast: [recipeId], lunch: [], dinner: [] }, ... }
  const [weeklyPlan, setWeeklyPlan] = useState(INITIAL_PLAN);

  // 週2回の買い出し設定: splitIndex は第1回買い出しの最終曜日インデックス (0:月 〜 6:日)
  // デフォルト 3 = 第1回：月〜木（4日分）、第2回：金〜日（3日分）
  const [splitDayIndex, setSplitDayIndex] = useState(3);
  const [activeShoppingTrip, setActiveShoppingTrip] = useState(1); // 1: 第1回買い出し, 2: 第2回買い出し
  const [checkedItems, setCheckedItems] = useState({}); // 買い物リストのチェック状態
  const [excludePantryStaples, setExcludePantryStaples] = useState(false); // 基本調味料をリストから隠すオプション

  // モーダル・UI状態
  const [selectedSlot, setSelectedSlot] = useState(null); // { dayKey, mealKey } 献立追加モーダル用
  const [inspectRecipeId, setInspectRecipeId] = useState(null); // 分量詳細・個別調味料微調整モーダル用
  const [showFamilyDrawer, setShowFamilyDrawer] = useState(true); // パパ・ママ・ジョウ・ソウの4名量レバーパネル（初期表示ON）
  const [showGiftBanner, setShowGiftBanner] = useState(true); // プレゼントメッセージバナー
  const [toastMessage, setToastMessage] = useState(null);

  // AI解析・新規レシピ／定番登録フォーム用ステート
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [recipeFormMode, setRecipeFormMode] = useState('ai'); // 'ai' | 'manual'
  const [aiInputText, setAiInputText] = useState('');
  const [manualTitle, setManualTitle] = useState('');
  const [manualType, setManualType] = useState('preset'); // 'preset' | 'video'
  const [manualServings, setManualServings] = useState(3.5);
  const [manualUrl, setManualUrl] = useState('');
  const [manualIngredients, setManualIngredients] = useState([
    {
      id: 'm-init-1',
      name: '',
      amount: 1,
      unit: '個',
      category: '野菜・果物',
      isSeasoning: false,
      customMultiplier: 1,
    },
  ]);

  // 画面下部に一時的な通知メッセージを表示するヘルパー
  const showToast = (msg) => {
    setToastMessage(msg);
    setTimeout(() => {
      setToastMessage((prev) => (prev === msg ? null : prev));
    }, 3200);
  };

  // 1. 認証の初期化（保存領域へのアクセス準備）
  useEffect(() => {
    if (!auth) return;
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error('Auth initialization failed:', err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
    });
    return () => unsubscribe();
  }, []);

  // 2. 保存済みデータの読み込み＆リアルタイム同期
  useEffect(() => {
    if (!user || !db) return;

    const stateDocRef = doc(db, 'artifacts', appId, 'users', user.uid, 'plannerData', 'main');
    const unsubscribe = onSnapshot(
      stateDocRef,
      (snapshot) => {
        if (snapshot.exists()) {
          const data = snapshot.data();
          const normalized = {
            recipes: Array.isArray(data.recipes) && data.recipes.length > 0 ? data.recipes : INITIAL_RECIPES,
            weeklyPlan: data.weeklyPlan || INITIAL_PLAN,
            papaRatio: typeof data.papaRatio === 'number' ? data.papaRatio : 1.0,
            mamaRatio: typeof data.mamaRatio === 'number' ? data.mamaRatio : 1.0,
            joRatio: typeof data.joRatio === 'number' ? data.joRatio : 0.75,
            soRatio: typeof data.soRatio === 'number' ? data.soRatio : 0.75,
            seasoningRatio: typeof data.seasoningRatio === 'number' ? data.seasoningRatio : 0.8,
            splitDayIndex: typeof data.splitDayIndex === 'number' ? data.splitDayIndex : 3,
            checkedItems: data.checkedItems || {},
          };

          lastSavedJsonRef.current = JSON.stringify(normalized);
          setRecipes(normalized.recipes);
          setWeeklyPlan(normalized.weeklyPlan);
          setPapaRatio(normalized.papaRatio);
          setMamaRatio(normalized.mamaRatio);
          setJoRatio(normalized.joRatio);
          setSoRatio(normalized.soRatio);
          setSeasoningRatio(normalized.seasoningRatio);
          setSplitDayIndex(normalized.splitDayIndex);
          setCheckedItems(normalized.checkedItems);
        } else {
          // 初回アクセス時は現在の初期データをそのまま保存基準にする
          const initialSnapshot = {
            recipes: INITIAL_RECIPES,
            weeklyPlan: INITIAL_PLAN,
            papaRatio: 1.0,
            mamaRatio: 1.0,
            joRatio: 0.75,
            soRatio: 0.75,
            seasoningRatio: 0.8,
            splitDayIndex: 3,
            checkedItems: {},
          };
          lastSavedJsonRef.current = JSON.stringify(initialSnapshot);
          setDoc(stateDocRef, initialSnapshot).catch((err) =>
            console.error('Initial save failed:', err)
          );
        }
        isRemoteLoadedRef.current = true;
        setSyncStatus('saved');
      },
      (error) => {
        console.error('Data sync error:', error);
      }
    );

    return () => unsubscribe();
  }, [user]);

  // 3. 献立・マイレシピ・各レバー変更時の自動保存
  useEffect(() => {
    if (!user || !db || !isRemoteLoadedRef.current) return;

    const currentPayload = {
      recipes,
      weeklyPlan,
      papaRatio,
      mamaRatio,
      joRatio,
      soRatio,
      seasoningRatio,
      splitDayIndex,
      checkedItems,
    };

    const serialized = JSON.stringify(currentPayload);
    if (serialized === lastSavedJsonRef.current) return;

    setSyncStatus('saving');
    const timer = setTimeout(async () => {
      try {
        const stateDocRef = doc(db, 'artifacts', appId, 'users', user.uid, 'plannerData', 'main');
        await setDoc(stateDocRef, currentPayload);
        lastSavedJsonRef.current = serialized;
        setSyncStatus('saved');
      } catch (err) {
        console.error('Auto-save failed:', err);
      }
    }, 450);

    return () => clearTimeout(timer);
  }, [
    user,
    recipes,
    weeklyPlan,
    papaRatio,
    mamaRatio,
    joRatio,
    soRatio,
    seasoningRatio,
    splitDayIndex,
    checkedItems,
  ]);

  // 家族4人合計の必要人前（例: パパ1.0 + ママ1.0 + じょう0.75 + そう0.75 = 3.5人前）
  const totalFamilyServings = useMemo(() => {
    return Math.round((papaRatio + mamaRatio + joRatio + soRatio) * 100) / 100;
  }, [papaRatio, mamaRatio, joRatio, soRatio]);

  // レシピ内の各材料の「わが家仕様の実際の必要量」を計算する関数
  const calculateScaledIngredient = (recipe, ingredient) => {
    const baseScale = totalFamilyServings / (recipe.servings || 1);
    const seasoningScale = ingredient.isSeasoning ? seasoningRatio : 1.0;
    const itemCustomScale = ingredient.customMultiplier ?? 1.0;
    const scaledAmount = ingredient.amount * baseScale * seasoningScale * itemCustomScale;
    return {
      scaledAmount,
      formattedAmount: formatAmount(scaledAmount),
    };
  };

  // 子供（じょう・そうの大人比パーセンテージ反映）が「主菜＋主食＋スープ＋チーズ＋果物」を食べた時の栄養素と充足率を計算
  const calculateChildNutritionForRecipe = (
    recipe,
    basisMode = 'meal',
    options = {
      stapleId: 'none',
      withHarvardSoup: false,
      withQbbCheese: false,
      fruitId: 'none',
      childRatio: CHILD_BASE_ADULT_RATIO,
    }
  ) => {
    const baseNutr = recipe?.nutritionPerServing || {
      calories: 380,
      protein: 15,
      calcium: 80,
      iron: 1.5,
      fiber: 2.5,
      salt: 1.8,
    };

    // 大人100%基準に対する子供の食べる量割合（デフォルト 0.75 = 75%）
    const childAdultRatio = options.childRatio ?? CHILD_BASE_ADULT_RATIO;
    // 主食・ハーバードスープ・果物の基準データ（75%基準）に対する相対倍率
    const relativeToStandardChild = childAdultRatio / CHILD_BASE_ADULT_RATIO;

    const stapleObj =
      STAPLE_OPTIONS.find((s) => s.id === options.stapleId) || STAPLE_OPTIONS[0];
    const soupNutr = options.withHarvardSoup
      ? HARVARD_SOUP_OPTION.nutrition
      : { calories: 0, protein: 0, calcium: 0, iron: 0, fiber: 0, salt: 0 };
    const cheeseNutr = options.withQbbCheese
      ? QBB_CHEESE_OPTION.nutrition
      : { calories: 0, protein: 0, calcium: 0, iron: 0, fiber: 0, salt: 0 };
    const fruitObj =
      DESSERT_FRUIT_OPTIONS.find((f) => f.id === options.fruitId) || DESSERT_FRUIT_OPTIONS[0];

    const divisor = basisMode === 'meal' ? 3 : 1; // 1食目安は1日の1/3

    const results = {};
    Object.entries(CHILD_DAILY_NUTRITION_TARGETS).forEach(([key, meta]) => {
      // 主菜の塩分だけは上部の「調味料の濃さレバー(seasoningRatio)」に連動させる
      const mainDishMultiplier =
        key === 'salt' ? childAdultRatio * seasoningRatio : childAdultRatio;
      const mainDishVal = (baseNutr[key] || 0) * mainDishMultiplier;
      // 主食・ハーバードスープ・果物には子供の食べる量レバー割合を反映（チーズは1個単位で合算）
      const stapleVal = (stapleObj.nutrition[key] || 0) * relativeToStandardChild;
      const soupVal = (soupNutr[key] || 0) * relativeToStandardChild;
      const cheeseVal = cheeseNutr[key] || 0;
      const fruitVal = (fruitObj.nutrition[key] || 0) * relativeToStandardChild;

      const actualValue = mainDishVal + stapleVal + soupVal + cheeseVal + fruitVal;
      const targetValue = meta.daily / divisor;
      const percentage = Math.round((actualValue / targetValue) * 100);

      let status = 'ok'; // 'low' | 'ok' | 'great' | 'high'
      if (meta.isUpperLimit) {
        status = percentage > 105 ? 'high' : 'ok';
      } else {
        if (percentage < 70) status = 'low';
        else if (percentage >= 95) status = 'great';
        else status = 'ok';
      }

      results[key] = {
        ...meta,
        actual: Math.round(actualValue * 10) / 10,
        target: Math.round(targetValue * 10) / 10,
        percentage,
        status,
      };
    });

    return results;
  };

  // URL栄養分析タブからのワンステップ解析＆即表示ハンドラ
  const handleQuickUrlAnalyze = async (customPromptText) => {
    const targetText = (customPromptText ?? quickUrlInput).trim();
    if (!targetText) {
      showToast('レシピ動画のURLや料理名・材料を貼り付けてください');
      return;
    }
    setIsAnalyzing(true);
    try {
      const extracted = await extractRecipeWithGemini(targetText);
      const urlMatch = targetText.match(/https?:\/\/[^\s]+/);

      const newRecipe = {
        id: `rec-${Date.now()}`,
        title: extracted.title || '解析した動画レシピ',
        type: 'video',
        servings: Number(extracted.servings) || 2,
        sourceUrl: urlMatch ? urlMatch[0] : '',
        tags: ['動画レシピ', 'AI栄養分析済'],
        nutritionPerServing: extracted.nutritionPerServing || {
          calories: 400,
          protein: 16,
          calcium: 70,
          iron: 1.6,
          fiber: 3.0,
          salt: 2.0,
        },
        childNutritionAdvice:
          extracted.childNutritionAdvice ||
          '野菜や乳製品を1品添えると、3〜4歳のお子さんに必要なカルシウム・鉄分・食物繊維がよりバランスよく補えます。',
        ingredients: (extracted.ingredients || []).map((item, idx) => ({
          id: `ai-${Date.now()}-${idx}`,
          name: item.name,
          amount: Number(item.amount) || 1,
          unit: item.unit || '個',
          category: INGREDIENT_CATEGORIES.includes(item.category)
            ? item.category
            : '乾物・米・その他',
          isSeasoning: Boolean(item.isSeasoning),
          customMultiplier: 1.0,
        })),
      };

      setRecipes((prev) => [newRecipe, ...prev]);
      setSelectedAnalyzeRecipeId(newRecipe.id);
      setQuickUrlInput('');
      showToast(`「${newRecipe.title}」の子供栄養分析＆分量換算が完了しました！`);
    } catch (error) {
      showToast('解析に失敗しました。料理名や材料テキストを含めてもう一度お試しください');
    } finally {
      setIsAnalyzing(false);
    }
  };

  // 献立枠にレシピを追加
  const handleAddRecipeToSlot = (dayKey, mealKey, recipeId) => {
    setWeeklyPlan((prev) => ({
      ...prev,
      [dayKey]: {
        ...prev[dayKey],
        [mealKey]: [...prev[dayKey][mealKey], recipeId],
      },
    }));
    setSelectedSlot(null);
    const recipe = recipes.find((r) => r.id === recipeId);
    showToast(`「${recipe?.title || 'メニュー'}」を献立に追加しました`);
  };

  // 献立枠からレシピを削除
  const handleRemoveRecipeFromSlot = (dayKey, mealKey, indexToRemove) => {
    setWeeklyPlan((prev) => ({
      ...prev,
      [dayKey]: {
        ...prev[dayKey],
        [mealKey]: prev[dayKey][mealKey].filter((_, idx) => idx !== indexToRemove),
      },
    }));
  };

  // 定番朝食を月〜日の朝に一括配置する便利機能
  const handleBulkFillBreakfast = (recipeId, mode = 'weekdays') => {
    const targetDays = mode === 'weekdays' ? ['mon', 'tue', 'wed', 'thu', 'fri'] : DAYS.map((d) => d.key);
    setWeeklyPlan((prev) => {
      const next = { ...prev };
      targetDays.forEach((dayKey) => {
        next[dayKey] = {
          ...next[dayKey],
          breakfast: [recipeId],
        };
      });
      return next;
    });
    const recipe = recipes.find((r) => r.id === recipeId);
    showToast(
      `${mode === 'weekdays' ? '平日（月〜金）' : '毎日（月〜日）'}の朝に「${recipe?.title}」を設定しました！`
    );
  };

  // レシピ内の個別材料の倍率（customMultiplier）を微増減する
  const handleAdjustIngredientMultiplier = (recipeId, ingredientId, delta) => {
    setRecipes((prev) =>
      prev.map((rec) => {
        if (rec.id !== recipeId) return rec;
        return {
          ...rec,
          ingredients: rec.ingredients.map((ing) => {
            if (ing.id !== ingredientId) return ing;
            const current = ing.customMultiplier ?? 1.0;
            const nextVal = Math.max(0.2, Math.min(2.5, Math.round((current + delta) * 10) / 10));
            return { ...ing, customMultiplier: nextVal };
          }),
        };
      })
    );
  };

  // AIでレシピ動画URL・テキストを解析してフォームにセット（または即登録）
  const handleAiExtract = async () => {
    if (!aiInputText.trim()) {
      showToast('レシピ動画のURLや料理名、材料テキストを入力してください');
      return;
    }
    setIsAnalyzing(true);
    try {
      const extracted = await extractRecipeWithGemini(aiInputText.trim());
      setManualTitle(extracted.title || '新しい動画レシピ');
      setManualType('video');
      setManualServings(extracted.servings || 2);
      // URLが含まれていれば抽出
      const urlMatch = aiInputText.match(/https?:\/\/[^\s]+/);
      setManualUrl(urlMatch ? urlMatch[0] : '');

      const mappedIngredients = (extracted.ingredients || []).map((item, idx) => ({
        id: `ai-${Date.now()}-${idx}`,
        name: item.name,
        amount: Number(item.amount) || 1,
        unit: item.unit || '個',
        category: INGREDIENT_CATEGORIES.includes(item.category) ? item.category : '乾物・米・その他',
        isSeasoning: Boolean(item.isSeasoning),
        customMultiplier: 1.0,
      }));

      setManualIngredients(
        mappedIngredients.length > 0
          ? mappedIngredients
          : [{ id: `m-${Date.now()}`, name: '', amount: 1, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 }]
      );
      setRecipeFormMode('manual');
      showToast('AIがレシピと何人前かを解析しました！内容を確認して保存してください');
    } catch (error) {
      showToast('解析に失敗しました。手動入力フォームから登録することもできます');
    } finally {
      setIsAnalyzing(false);
    }
  };

  // 新規レシピ・定番フォーマットの保存
  const handleSaveNewRecipe = (e) => {
    e.preventDefault();
    if (!manualTitle.trim()) {
      showToast('料理名・セット名を入力してください');
      return;
    }
    const validIngredients = manualIngredients.filter((ing) => ing.name.trim() !== '');
    if (validIngredients.length === 0) {
      showToast('材料を1つ以上入力してください');
      return;
    }

    const newRecipe = {
      id: `rec-${Date.now()}`,
      title: manualTitle.trim(),
      type: manualType,
      servings: Number(manualServings) || 2,
      sourceUrl: manualUrl.trim(),
      tags: manualType === 'video' ? ['動画レシピ'] : ['定番・個別'],
      nutritionPerServing: {
        calories: 380,
        protein: 15,
        calcium: 90,
        iron: 1.6,
        fiber: 3.0,
        salt: 1.6,
      },
      childNutritionAdvice:
        '手動登録されたメニューです。牛乳や小魚、緑黄色野菜を副菜に組み合わせるとお子様の成長に必要な栄養素がさらに充実します。',
      ingredients: validIngredients.map((ing, i) => ({
        ...ing,
        id: ing.id || `ing-${Date.now()}-${i}`,
        amount: Number(ing.amount) || 0,
        customMultiplier: 1.0,
      })),
    };

    setRecipes((prev) => [newRecipe, ...prev]);
    setSelectedAnalyzeRecipeId(newRecipe.id);
    // フォームリセット
    setManualTitle('');
    setManualUrl('');
    setAiInputText('');
    setManualIngredients([
      { id: `m-${Date.now()}`, name: '', amount: 1, unit: '個', category: '野菜・果物', isSeasoning: false, customMultiplier: 1 },
    ]);
    showToast(`「${newRecipe.title}」をマイレシピに保存しました！`);
  };

  // レシピ削除
  const handleDeleteRecipe = (recipeId) => {
    setRecipes((prev) => prev.filter((r) => r.id !== recipeId));
    // 献立からも該当レシピを除去
    setWeeklyPlan((prev) => {
      const next = {};
      for (const day of DAYS) {
        next[day.key] = {
          breakfast: prev[day.key].breakfast.filter((id) => id !== recipeId),
          lunch: prev[day.key].lunch.filter((id) => id !== recipeId),
          dinner: prev[day.key].dinner.filter((id) => id !== recipeId),
        };
      }
      return next;
    });
    if (inspectRecipeId === recipeId) setInspectRecipeId(null);
    showToast('レシピを削除しました');
  };

  // ============================================================================
  // 週2回の買い物リスト一括合算ロジック
  // ============================================================================

  const shoppingLists = useMemo(() => {
    const buildListForDays = (daySlice) => {
      const map = new Map();

      daySlice.forEach((dayObj) => {
        const dayMeals = weeklyPlan[dayObj.key];
        if (!dayMeals) return;

        ['breakfast', 'lunch', 'dinner'].forEach((mealKey) => {
          const recipeIds = dayMeals[mealKey] || [];
          recipeIds.forEach((rId) => {
            const recipe = recipes.find((r) => r.id === rId);
            if (!recipe) return;

            recipe.ingredients.forEach((ing) => {
              const scaled = calculateScaledIngredient(recipe, ing);
              // 食材名と単位を正規化して合算キーを作成
              const normalizedName = ing.name.trim();
              const normalizedUnit = ing.unit.trim();
              const key = `${normalizedName}__${normalizedUnit}`;

              if (map.has(key)) {
                const existing = map.get(key);
                existing.totalAmount += scaled.scaledAmount;
                if (!existing.usedIn.includes(recipe.title)) {
                  existing.usedIn.push(recipe.title);
                }
              } else {
                map.set(key, {
                  key,
                  name: normalizedName,
                  unit: normalizedUnit,
                  category: ing.category || '乾物・米・その他',
                  isSeasoning: ing.isSeasoning,
                  totalAmount: scaled.scaledAmount,
                  usedIn: [recipe.title],
                });
              }
            });
          });
        });
      });

      // カテゴリ別に整理
      const items = Array.from(map.values()).map((item) => ({
        ...item,
        formattedAmount: formatAmount(item.totalAmount),
      }));

      return items;
    };

    const trip1Days = DAYS.slice(0, splitDayIndex + 1);
    const trip2Days = DAYS.slice(splitDayIndex + 1);

    return {
      trip1: {
        days: trip1Days,
        label: `${trip1Days[0]?.label}〜${trip1Days[trip1Days.length - 1]?.label}曜日分`,
        items: buildListForDays(trip1Days),
      },
      trip2: {
        days: trip2Days,
        label:
          trip2Days.length > 0
            ? `${trip2Days[0]?.label}〜${trip2Days[trip2Days.length - 1]?.label}曜日分`
            : '該当なし',
        items: buildListForDays(trip2Days),
      },
    };
  }, [weeklyPlan, recipes, splitDayIndex, totalFamilyServings, seasoningRatio]);

  // 買い物リストをテキストコピー（LINE等で共有用）
  const handleCopyShoppingList = () => {
    const currentTrip = activeShoppingTrip === 1 ? shoppingLists.trip1 : shoppingLists.trip2;
    const filtered = currentTrip.items.filter(
      (item) => !(excludePantryStaples && item.isSeasoning)
    );

    if (filtered.length === 0) {
      showToast('コピーする買い物リストがありません');
      return;
    }

    let text = `🛒【第${activeShoppingTrip}回 買い物リスト（${currentTrip.label}）】\n`;
    text += `家族設定: パパ${Math.round(papaRatio * 100)}%・ママ${Math.round(
      mamaRatio * 100
    )}%・じょう${Math.round(joRatio * 100)}%・そう${Math.round(
      soRatio * 100
    )}%（計${totalFamilyServings}人前 / 調味料${Math.round(seasoningRatio * 100)}%）\n\n`;

    INGREDIENT_CATEGORIES.forEach((cat) => {
      const catItems = filtered.filter((i) => i.category === cat);
      if (catItems.length > 0) {
        text += `■ ${cat}\n`;
        catItems.forEach((item) => {
          const checkMark = checkedItems[`${activeShoppingTrip}-${item.key}`] ? '☑' : '☐';
          text += `${checkMark} ${item.name}：${item.formattedAmount}${item.unit}\n`;
        });
        text += '\n';
      }
    });

    navigator.clipboard?.writeText(text);
    showToast('買い物リストをクリップボードにコピーしました！');
  };

  // 現在詳細表示中のレシピオブジェクト
  const inspectedRecipe = useMemo(
    () => recipes.find((r) => r.id === inspectRecipeId) || null,
    [recipes, inspectRecipeId]
  );

  // URL栄養分析タブで選択中のレシピオブジェクト
  const analyzedRecipe = useMemo(
    () => recipes.find((r) => r.id === selectedAnalyzeRecipeId) || recipes[0] || null,
    [recipes, selectedAnalyzeRecipeId]
  );

  // ============================================================================
  // レンダリング
  // ============================================================================

  return (
    <div className="min-h-screen bg-[#FAF8F5] text-stone-800 font-sans pb-24">
      {/* トースト通知 */}
      {toastMessage && (
        <div className="fixed bottom-20 left-1/2 -translate-x-1/2 z-50 bg-stone-800 text-white px-5 py-3 rounded-full shadow-xl flex items-center gap-2.5 text-sm font-medium animate-fade-in max-w-[92vw]">
          <Check className="w-4 h-4 text-emerald-400 shrink-0" />
          <span>{toastMessage}</span>
        </div>
      )}

      {/* ヘッダー */}
      <header className="bg-white border-b border-stone-200/80 sticky top-0 z-30 shadow-xs">
        <div className="max-w-6xl mx-auto px-4 py-3 flex flex-wrap items-center justify-between gap-3">
          <div className="flex items-center gap-2.5">
            <div className="w-10 h-10 rounded-2xl bg-gradient-to-br from-orange-500 to-amber-500 flex items-center justify-center text-white shadow-sm">
              <ChefHat className="w-6 h-6" />
            </div>
            <div>
              <h1 className="text-lg sm:text-xl font-bold tracking-tight text-stone-800 flex items-center gap-1.5">
                髙橋紘太の栄養レシピ
              </h1>
              <p className="text-xs text-stone-500">
                家族4人分自動計算・薄味調整・週2回まとめ買い
              </p>
            </div>
          </div>

          {/* 家族人数＆薄味レバー クイックコントローラー */}
          <div className="flex items-center gap-2 sm:gap-3 flex-wrap">
            {/* 自動保存ステータスバッジ */}
            <div
              className={`flex items-center gap-1.5 px-2.5 py-1.5 rounded-xl border text-xs font-bold transition ${
                syncStatus === 'saving'
                  ? 'bg-amber-50 border-amber-200 text-amber-700'
                  : syncStatus === 'saved'
                  ? 'bg-emerald-50 border-emerald-200 text-emerald-700'
                  : 'bg-stone-100 border-stone-200 text-stone-500'
              }`}
              title="登録したレシピや献立・レバー設定は自動的に保存されます"
            >
              {syncStatus === 'saving' ? (
                <>
                  <RefreshCw className="w-3.5 h-3.5 animate-spin" />
                  <span>保存中...</span>
                </>
              ) : syncStatus === 'saved' ? (
                <>
                  <Cloud className="w-3.5 h-3.5 text-emerald-600" />
                  <span>自動保存済み</span>
                </>
              ) : (
                <>
                  <Cloud className="w-3.5 h-3.5" />
                  <span>接続中...</span>
                </>
              )}
            </div>

            <button
              onClick={() => setShowFamilyDrawer(!showFamilyDrawer)}
              className="flex items-center gap-2 px-3 py-1.5 rounded-xl bg-orange-50 hover:bg-orange-100/80 border border-orange-200/80 text-orange-900 text-xs sm:text-sm font-medium transition cursor-pointer"
            >
              <Users className="w-4 h-4 text-orange-600" />
              <span>
                計 <strong>{totalFamilyServings}</strong> 人前
              </span>
              <span className="text-orange-400">|</span>
              <Sliders className="w-3.5 h-3.5 text-orange-600" />
              <span>
                味の濃さ <strong>{Math.round(seasoningRatio * 100)}%</strong>
              </span>
            </button>
          </div>
        </div>

        {/* 味の濃さレバー ＆ 家族分量調整バー（常時アクセスしやすいコントロールバー） */}
        <div className="bg-gradient-to-r from-amber-50/90 via-orange-50/70 to-emerald-50/70 border-t border-stone-200/60">
          <div className="max-w-6xl mx-auto px-4 py-2.5 flex flex-col md:flex-row md:items-center justify-between gap-3">
            {/* 味の濃さ調整レバー（スライダー） */}
            <div className="flex-1 flex flex-col sm:flex-row sm:items-center gap-2 sm:gap-4">
              <div className="flex items-center gap-1.5 shrink-0">
                <span className="inline-flex items-center justify-center w-6 h-6 rounded-full bg-orange-500 text-white text-xs font-bold">
                  味
                </span>
                <span className="text-xs sm:text-sm font-bold text-stone-700">
                  調味料の濃さレバー:
                </span>
                <span
                  className={`px-2 py-0.5 rounded-md text-xs font-bold ${
                    seasoningRatio < 0.85
                      ? 'bg-emerald-100 text-emerald-800'
                      : seasoningRatio > 1.0
                      ? 'bg-rose-100 text-rose-800'
                      : 'bg-amber-100 text-amber-800'
                  }`}
                >
                  {Math.round(seasoningRatio * 100)}%
                  {seasoningRatio <= 0.75
                    ? '（しっかり薄味）'
                    : seasoningRatio <= 0.9
                    ? '（子供向けやさしめ）'
                    : seasoningRatio === 1.0
                    ? '（レシピ通り）'
                    : '（しっかり味）'}
                </span>
              </div>

              <div className="flex items-center gap-2.5 flex-1 max-w-md">
                <span className="text-[11px] font-medium text-emerald-700 shrink-0">薄め 50%</span>
                <input
                  type="range"
                  min="0.5"
                  max="1.2"
                  step="0.05"
                  value={seasoningRatio}
                  onChange={(e) => setSeasoningRatio(parseFloat(e.target.value))}
                  className="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-orange-500"
                  aria-label="調味料の濃さ調整レバー"
                />
                <span className="text-[11px] font-medium text-stone-500 shrink-0">濃め 120%</span>
              </div>
            </div>

            {/* 家族割合サマリー＆詳細設定トグル */}
            <div className="flex items-center justify-between md:justify-end gap-3 text-xs text-stone-600">
              <span className="bg-white/80 px-2.5 py-1 rounded-lg border border-stone-200/80">
                パパ{Math.round(papaRatio * 100)}%・ママ{Math.round(mamaRatio * 100)}%・じょう
                {Math.round(joRatio * 100)}%・そう{Math.round(soRatio * 100)}%
              </span>
              <button
                onClick={() => setShowFamilyDrawer(!showFamilyDrawer)}
                className="text-orange-600 hover:text-orange-700 font-semibold underline underline-offset-2 shrink-0 cursor-pointer"
              >
                {showFamilyDrawer ? '4人の量レバーを閉じる' : '4人の量レバーを開く'}
              </button>
            </div>
          </div>

          {/* 展開式：パパ・ママ・じょう・そうの4名個別「食べる量レバー」設定 */}
          {showFamilyDrawer && (
            <div className="max-w-6xl mx-auto px-4 pb-3.5 pt-2 border-t border-orange-200/50 space-y-2">
              <div className="flex flex-wrap items-center justify-between gap-2 text-[11px] text-stone-600">
                <span className="font-bold text-stone-700">
                  👨‍👩‍👦‍👦 家族4人の食べる量レバー（全員「大人100%」を基準に調整・じょう＆そうの初期値は75%）
                </span>
                <button
                  type="button"
                  onClick={() => {
                    setPapaRatio(1.0);
                    setMamaRatio(1.0);
                    setJoRatio(0.75);
                    setSoRatio(0.75);
                    showToast('4人の食べる量を初期値（パパ100%・ママ100%・じょう75%・そう75%）に戻しました');
                  }}
                  className="text-stone-500 hover:text-orange-600 font-bold underline cursor-pointer"
                >
                  初期値にリセット
                </button>
              </div>

              <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-2.5">
                {/* パパ レバー */}
                <div className="bg-white p-3 rounded-xl border border-stone-200 space-y-1.5">
                  <div className="flex items-center justify-between">
                    <div>
                      <span className="text-xs font-extrabold text-stone-800">パパ</span>
                      <span className="ml-1.5 text-[10px] text-stone-400">大人基準100%</span>
                    </div>
                    <span className="text-xs font-extrabold px-2 py-0.5 rounded-md bg-sky-100 text-sky-800">
                      {Math.round(papaRatio * 100)}%
                    </span>
                  </div>
                  <input
                    type="range"
                    min="0.6"
                    max="1.5"
                    step="0.05"
                    value={papaRatio}
                    onChange={(e) => setPapaRatio(parseFloat(e.target.value))}
                    className="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-sky-600"
                    aria-label="パパの食べる量レバー"
                  />
                  <div className="flex justify-between text-[10px] text-stone-400">
                    <span>60%</span>
                    <span>基準 100%</span>
                    <span>150%</span>
                  </div>
                </div>

                {/* ママ レバー */}
                <div className="bg-white p-3 rounded-xl border border-stone-200 space-y-1.5">
                  <div className="flex items-center justify-between">
                    <div>
                      <span className="text-xs font-extrabold text-stone-800">ママ</span>
                      <span className="ml-1.5 text-[10px] text-stone-400">大人基準100%</span>
                    </div>
                    <span className="text-xs font-extrabold px-2 py-0.5 rounded-md bg-rose-100 text-rose-800">
                      {Math.round(mamaRatio * 100)}%
                    </span>
                  </div>
                  <input
                    type="range"
                    min="0.6"
                    max="1.5"
                    step="0.05"
                    value={mamaRatio}
                    onChange={(e) => setMamaRatio(parseFloat(e.target.value))}
                    className="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-rose-500"
                    aria-label="ママの食べる量レバー"
                  />
                  <div className="flex justify-between text-[10px] text-stone-400">
                    <span>60%</span>
                    <span>基準 100%</span>
                    <span>150%</span>
                  </div>
                </div>

                {/* じょう（4歳） レバー */}
                <div className="bg-white p-3 rounded-xl border border-orange-200/80 space-y-1.5">
                  <div className="flex items-center justify-between">
                    <div>
                      <span className="text-xs font-extrabold text-stone-800">じょう（4歳）</span>
                      <span className="ml-1.5 text-[10px] text-stone-400">大人100%基準</span>
                    </div>
                    <span className="text-xs font-extrabold px-2 py-0.5 rounded-md bg-orange-100 text-orange-800">
                      {Math.round(joRatio * 100)}%
                    </span>
                  </div>
                  <input
                    type="range"
                    min="0.4"
                    max="1.2"
                    step="0.05"
                    value={joRatio}
                    onChange={(e) => setJoRatio(parseFloat(e.target.value))}
                    className="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-orange-500"
                    aria-label="じょうの食べる量レバー"
                  />
                  <div className="flex justify-between text-[10px] text-stone-400">
                    <span>40%</span>
                    <span>初期 75%</span>
                    <span>基準 100%</span>
                  </div>
                </div>

                {/* そう（3歳） レバー */}
                <div className="bg-white p-3 rounded-xl border border-emerald-200/80 space-y-1.5">
                  <div className="flex items-center justify-between">
                    <div>
                      <span className="text-xs font-extrabold text-stone-800">そう（3歳）</span>
                      <span className="ml-1.5 text-[10px] text-stone-400">大人100%基準</span>
                    </div>
                    <span className="text-xs font-extrabold px-2 py-0.5 rounded-md bg-emerald-100 text-emerald-800">
                      {Math.round(soRatio * 100)}%
                    </span>
                  </div>
                  <input
                    type="range"
                    min="0.4"
                    max="1.2"
                    step="0.05"
                    value={soRatio}
                    onChange={(e) => setSoRatio(parseFloat(e.target.value))}
                    className="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-emerald-600"
                    aria-label="そうの食べる量レバー"
                  />
                  <div className="flex justify-between text-[10px] text-stone-400">
                    <span>40%</span>
                    <span>初期 75%</span>
                    <span>基準 100%</span>
                  </div>
                </div>
              </div>
            </div>
          )}
        </div>
      </header>

      {/* メインコンテンツエリア */}
      <main className="max-w-6xl mx-auto px-4 pt-5">
        {/* プレゼントメッセージカード（閉じられる） */}
        {showGiftBanner && (
          <div className="mb-5 bg-gradient-to-r from-orange-500 to-amber-500 text-white rounded-2xl p-4 shadow-sm flex items-start justify-between gap-3">
            <div className="flex items-start gap-3">
              <div className="w-9 h-9 rounded-xl bg-white/20 flex items-center justify-center shrink-0 mt-0.5">
                <Heart className="w-5 h-5 fill-white text-white" />
              </div>
              <div className="text-xs sm:text-sm leading-relaxed">
                <p className="font-bold text-sm sm:text-base mb-0.5">
                  いつも美味しいごはんとお買い物をありがとう！
                </p>
                <p className="text-orange-50">
                  レシピ動画を入れるだけで「大人2人＋4歳・3歳（計{totalFamilyServings}人前）」の分量に自動換算＆上のレバーで子供向けの薄味調整ができます。週2回の買い物リストも自動で作れるよ！
                </p>
              </div>
            </div>
            <button
              onClick={() => setShowGiftBanner(false)}
              className="p-1.5 rounded-lg hover:bg-white/20 text-white/80 hover:text-white transition shrink-0 cursor-pointer"
              aria-label="メッセージを閉じる"
            >
              <X className="w-4 h-4" />
            </button>
          </div>
        )}

        {/* メインナビゲーションタブ（PC/スマホ共通の上部ピルタブ） */}
        <div className="grid grid-cols-2 sm:grid-cols-4 gap-2 bg-stone-200/70 p-1.5 rounded-2xl mb-6">
          <button
            onClick={() => setActiveTab('analyze')}
            className={`flex items-center justify-center gap-1.5 py-2.5 px-3 rounded-xl text-xs sm:text-sm font-bold transition cursor-pointer ${
              activeTab === 'analyze'
                ? 'bg-white text-orange-600 shadow-sm'
                : 'text-stone-600 hover:text-stone-800'
            }`}
          >
            <Activity className="w-4 h-4 shrink-0" />
            <span>URLで栄養＆分量</span>
          </button>

          <button
            onClick={() => setActiveTab('calendar')}
            className={`flex items-center justify-center gap-1.5 py-2.5 px-3 rounded-xl text-xs sm:text-sm font-bold transition cursor-pointer ${
              activeTab === 'calendar'
                ? 'bg-white text-orange-600 shadow-sm'
                : 'text-stone-600 hover:text-stone-800'
            }`}
          >
            <Calendar className="w-4 h-4 shrink-0" />
            <span>1週間の献立表</span>
          </button>

          <button
            onClick={() => setActiveTab('recipes')}
            className={`flex items-center justify-center gap-1.5 py-2.5 px-3 rounded-xl text-xs sm:text-sm font-bold transition cursor-pointer ${
              activeTab === 'recipes'
                ? 'bg-white text-orange-600 shadow-sm'
                : 'text-stone-600 hover:text-stone-800'
            }`}
          >
            <BookOpen className="w-4 h-4 shrink-0" />
            <span>定番・個別登録</span>
          </button>

          <button
            onClick={() => setActiveTab('shopping')}
            className={`flex items-center justify-center gap-1.5 py-2.5 px-3 rounded-xl text-xs sm:text-sm font-bold transition cursor-pointer ${
              activeTab === 'shopping'
                ? 'bg-white text-emerald-700 shadow-sm'
                : 'text-stone-600 hover:text-stone-800'
            }`}
          >
            <ShoppingCart className="w-4 h-4 shrink-0" />
            <span>
              買い物リスト
              <span className="ml-1 inline-flex items-center justify-center px-1.5 py-0.5 text-[10px] rounded-full bg-emerald-100 text-emerald-800">
                {shoppingLists.trip1.items.length + shoppingLists.trip2.items.length}
              </span>
            </span>
          </button>
        </div>

        {/* ====================================================================
            タブ0（メイン）: レシピURLを貼るだけで子供の栄養分析＆わが家分量チェック
           ==================================================================== */}
        {activeTab === 'analyze' && (
          <div className="space-y-6">
            {/* 上部：URL・レシピ即解析カード */}
            <div className="bg-white rounded-3xl border border-orange-200/90 p-5 sm:p-6 shadow-xs">
              <div className="flex flex-col md:flex-row md:items-center justify-between gap-2 mb-3">
                <div>
                  <span className="inline-flex items-center gap-1 text-[11px] font-bold px-2.5 py-0.5 rounded-full bg-orange-100 text-orange-800 mb-1.5">
                    <Sparkles className="w-3 h-3" />
                    1週間分を組まなくても、レシピ1つからすぐ使える！
                  </span>
                  <h2 className="text-base sm:text-lg font-extrabold text-stone-800">
                    レシピ動画のURL・料理名を入れて、子供の栄養バランス＆わが家分量を診断
                  </h2>
                </div>
              </div>

              <div className="flex flex-col sm:flex-row gap-2.5">
                <input
                  type="text"
                  value={quickUrlInput}
                  onChange={(e) => setQuickUrlInput(e.target.value)}
                  onKeyDown={(e) => {
                    if (e.key === 'Enter') handleQuickUrlAnalyze();
                  }}
                  placeholder="レシピ動画のURL（YouTube/TikTok/Instagram等）や料理名・材料を貼り付け..."
                  className="flex-1 rounded-2xl border border-stone-300 px-4 py-3 text-sm focus:outline-none focus:ring-2 focus:ring-orange-500/30 focus:border-orange-500 bg-stone-50/50"
                />
                <button
                  onClick={() => handleQuickUrlAnalyze()}
                  disabled={isAnalyzing}
                  className="px-6 py-3 rounded-2xl bg-gradient-to-r from-orange-500 to-amber-500 hover:from-orange-600 hover:to-amber-600 disabled:opacity-60 text-white font-bold text-sm shadow-sm flex items-center justify-center gap-2 shrink-0 transition cursor-pointer"
                >
                  {isAnalyzing ? (
                    <>
                      <RefreshCw className="w-4 h-4 animate-spin" />
                      <span>AIが栄養＆分量を分析中...</span>
                    </>
                  ) : (
                    <>
                      <Activity className="w-4 h-4" />
                      <span>子供の栄養＆分量を分析</span>
                    </>
                  )}
                </button>
              </div>

              {/* ワンクリックお試しサンプルチップ */}
              <div className="mt-3 flex flex-wrap items-center gap-2 text-xs">
                <span className="text-stone-400 font-medium">お試しサンプルURL例:</span>
                {[
                  {
                    label: '鮭とほうれん草のクリームシチュー（2人前）',
                    query:
                      'https://www.youtube.com/watch?v=sample_salmon_stew 鮭とほうれん草のクリームシチュー 2人前 生鮭2切れ、ほうれん草1/2束、じゃがいも2個、玉ねぎ1/2個、牛乳300ml、バター15g、薄力粉大さじ2、コンソメ小さじ2',
                  },
                  {
                    label: '豚こまとピーマンの青椒肉絲（2人前）',
                    query:
                      'https://www.youtube.com/watch?v=sample_chinjao 豚こまとピーマンの青椒肉絲 2人前 豚こま肉200g、ピーマン4個、たけのこ水煮100g、オイスターソース大さじ1.5、醤油大さじ1、酒大さじ1、砂糖小さじ1',
                  },
                ].map((sample, i) => (
                  <button
                    key={i}
                    type="button"
                    disabled={isAnalyzing}
                    onClick={() => handleQuickUrlAnalyze(sample.query)}
                    className="px-2.5 py-1 rounded-lg bg-stone-100 hover:bg-orange-50 text-stone-600 hover:text-orange-700 border border-stone-200/80 transition cursor-pointer"
                  >
                    + {sample.label}
                  </button>
                ))}
              </div>
            </div>

            {/* 登録済みレシピ切り替えピルバー */}
            <div className="flex items-center gap-2 overflow-x-auto pb-1">
              <span className="text-xs font-bold text-stone-500 shrink-0">分析するレシピ:</span>
              {recipes.map((rec) => (
                <button
                  key={rec.id}
                  onClick={() => setSelectedAnalyzeRecipeId(rec.id)}
                  className={`px-3.5 py-2 rounded-xl text-xs font-bold shrink-0 border transition cursor-pointer ${
                    analyzedRecipe?.id === rec.id
                      ? 'bg-stone-800 text-white border-stone-800 shadow-2xs'
                      : 'bg-white text-stone-700 border-stone-200 hover:border-orange-300'
                  }`}
                >
                  {rec.title}
                </button>
              ))}
            </div>

            {/* 選択中レシピの「子供栄養分析ダッシュボード」＋「わが家分量・即買い物リスト」 */}
            {analyzedRecipe && (() => {
              const activeChildRatio =
                selectedChildForNutrition === 'jo' ? joRatio : soRatio;
              const activeChildName =
                selectedChildForNutrition === 'jo' ? 'じょう（4歳）' : 'そう（3歳）';

              const childNutr = calculateChildNutritionForRecipe(
                analyzedRecipe,
                nutritionBasisMode,
                {
                  stapleId: selectedStapleId,
                  withHarvardSoup: includeHarvardSoup,
                  withQbbCheese: includeQbbCheese,
                  fruitId: selectedFruitId,
                  childRatio: activeChildRatio,
                }
              );
              const missingNutrients = Object.values(childNutr).filter(
                (n) => !n.isUpperLimit && n.status === 'low'
              );
              const sufficientNutrients = Object.values(childNutr).filter(
                (n) => !n.isUpperLimit && (n.status === 'great' || n.status === 'ok')
              );
              const currentStaple =
                STAPLE_OPTIONS.find((s) => s.id === selectedStapleId) || STAPLE_OPTIONS[0];
              const currentFruit =
                DESSERT_FRUIT_OPTIONS.find((f) => f.id === selectedFruitId) ||
                DESSERT_FRUIT_OPTIONS[0];

              return (
                <div className="grid grid-cols-1 lg:grid-cols-12 gap-6">
                  {/* 左7カラム：子供栄養分析ダッシュボード */}
                  <div className="lg:col-span-7 bg-white rounded-3xl border border-stone-200/90 p-5 sm:p-6 shadow-2xs space-y-5">
                    <div className="flex flex-wrap items-start justify-between gap-3">
                      <div>
                        <span className="inline-flex items-center gap-1 text-[11px] font-bold px-2.5 py-0.5 rounded-full bg-emerald-100 text-emerald-800 mb-1">
                          <Sparkle className="w-3 h-3" />
                          {activeChildName}（大人比{Math.round(activeChildRatio * 100)}%）の栄養バランス判定
                        </span>
                        <h3 className="text-base sm:text-lg font-extrabold text-stone-800">
                          {analyzedRecipe.title}
                        </h3>
                      </div>

                      {/* 子供切り替え（じょう / そう）＆ 基準切り替え（1食目安 / 1日必要量） */}
                      <div className="flex flex-wrap items-center gap-2">
                        <div className="inline-flex rounded-xl bg-stone-100 p-1 text-xs font-bold">
                          <button
                            type="button"
                            onClick={() => setSelectedChildForNutrition('jo')}
                            className={`px-2.5 py-1 rounded-lg transition cursor-pointer ${
                              selectedChildForNutrition === 'jo'
                                ? 'bg-orange-500 text-white shadow-2xs'
                                : 'text-stone-600 hover:text-stone-800'
                            }`}
                          >
                            じょう ({Math.round(joRatio * 100)}%)
                          </button>
                          <button
                            type="button"
                            onClick={() => setSelectedChildForNutrition('so')}
                            className={`px-2.5 py-1 rounded-lg transition cursor-pointer ${
                              selectedChildForNutrition === 'so'
                                ? 'bg-emerald-600 text-white shadow-2xs'
                                : 'text-stone-600 hover:text-stone-800'
                            }`}
                          >
                            そう ({Math.round(soRatio * 100)}%)
                          </button>
                        </div>

                        <div className="inline-flex rounded-xl bg-stone-100 p-1 text-xs font-bold">
                          <button
                            type="button"
                            onClick={() => setNutritionBasisMode('meal')}
                            className={`px-2.5 py-1 rounded-lg transition cursor-pointer ${
                              nutritionBasisMode === 'meal'
                                ? 'bg-white text-stone-800 shadow-2xs'
                                : 'text-stone-500 hover:text-stone-800'
                            }`}
                          >
                            1食目安
                          </button>
                          <button
                            type="button"
                            onClick={() => setNutritionBasisMode('daily')}
                            className={`px-2.5 py-1 rounded-lg transition cursor-pointer ${
                              nutritionBasisMode === 'daily'
                                ? 'bg-white text-stone-800 shadow-2xs'
                                : 'text-stone-500 hover:text-stone-800'
                            }`}
                          >
                            1日基準
                          </button>
                        </div>
                      </div>
                    </div>

                    {/* 主食（炭水化物）・スープ・チーズ・デザート追加コントロールパネル */}
                    <div className="rounded-2xl bg-stone-50 border border-stone-200/80 p-3.5 space-y-3">
                      <div>
                        <div className="text-xs font-extrabold text-stone-700 mb-2 flex items-center justify-between">
                          <span>🍚 一緒に食べる主食（炭水化物）を選択：</span>
                          <span className="text-[11px] font-normal text-stone-500">
                            {currentStaple.id !== 'none'
                              ? `${activeChildName}の量: ${currentStaple.childPortionLabel}（大人比${Math.round(activeChildRatio * 100)}%）`
                              : '主菜のみで計算中'}
                          </span>
                        </div>
                        <div className="flex flex-wrap gap-1.5">
                          {STAPLE_OPTIONS.map((staple) => (
                            <button
                              key={staple.id}
                              type="button"
                              onClick={() => setSelectedStapleId(staple.id)}
                              className={`px-3 py-1.5 rounded-xl text-xs font-bold border transition cursor-pointer ${
                                selectedStapleId === staple.id
                                  ? 'bg-amber-500 text-white border-amber-500 shadow-2xs'
                                  : 'bg-white text-stone-700 border-stone-200 hover:border-amber-300'
                              }`}
                            >
                              {staple.label}
                            </button>
                          ))}
                        </div>
                      </div>

                      {/* ハーバードスープ追加トグル */}
                      <div className="pt-2.5 border-t border-stone-200/70 flex flex-wrap items-center justify-between gap-2">
                        <div className="flex items-center gap-2">
                          <span className="text-base">🥣</span>
                          <div>
                            <div className="text-xs font-extrabold text-stone-800">
                              わが家の「ハーバードスープ」をプラスする
                            </div>
                            <div className="text-[11px] text-stone-500">
                              塩分0gで食物繊維(+{(HARVARD_SOUP_OPTION.nutrition.fiber * (activeChildRatio / CHILD_BASE_ADULT_RATIO)).toFixed(1)}g)・カルシウム・鉄分を底上げ！
                            </div>
                          </div>
                        </div>

                        <button
                          type="button"
                          onClick={() => setIncludeHarvardSoup(!includeHarvardSoup)}
                          className={`px-3.5 py-1.5 rounded-xl text-xs font-extrabold border transition flex items-center gap-1.5 cursor-pointer ${
                            includeHarvardSoup
                              ? 'bg-emerald-600 text-white border-emerald-600 shadow-2xs'
                              : 'bg-white text-stone-600 border-stone-300 hover:border-emerald-400'
                          }`}
                        >
                          {includeHarvardSoup ? (
                            <>
                              <Check className="w-3.5 h-3.5" />
                              <span>追加中（1杯）</span>
                            </>
                          ) : (
                            <>
                              <Plus className="w-3.5 h-3.5" />
                              <span>スープを追加</span>
                            </>
                          )}
                        </button>
                      </div>

                      {/* Q・B・Bチーズ追加トグル */}
                      <div className="pt-2.5 border-t border-stone-200/70 flex flex-wrap items-center justify-between gap-2">
                        <div className="flex items-center gap-2">
                          <span className="text-base">🧀</span>
                          <div>
                            <div className="text-xs font-extrabold text-stone-800">
                              「Q・B・Bチーズ（5個入りパックの1個）」をプラスする
                            </div>
                            <div className="text-[11px] text-stone-500">
                              5個入りの1個でカルシウム(+95mg)・タンパク質(+3.2g)を手軽に補給！
                            </div>
                          </div>
                        </div>

                        <button
                          type="button"
                          onClick={() => setIncludeQbbCheese(!includeQbbCheese)}
                          className={`px-3.5 py-1.5 rounded-xl text-xs font-extrabold border transition flex items-center gap-1.5 cursor-pointer ${
                            includeQbbCheese
                              ? 'bg-amber-500 text-white border-amber-500 shadow-2xs'
                              : 'bg-white text-stone-600 border-stone-300 hover:border-amber-400'
                          }`}
                        >
                          {includeQbbCheese ? (
                            <>
                              <Check className="w-3.5 h-3.5" />
                              <span>追加中（1個）</span>
                            </>
                          ) : (
                            <>
                              <Plus className="w-3.5 h-3.5" />
                              <span>チーズを追加</span>
                            </>
                          )}
                        </button>
                      </div>

                      {/* デザート（果物）選択欄 */}
                      <div className="pt-2.5 border-t border-stone-200/70">
                        <div className="text-xs font-extrabold text-stone-700 mb-2 flex items-center justify-between">
                          <span>🍎 デザート（果物）を選択：</span>
                          <span className="text-[11px] font-normal text-stone-500">
                            {currentFruit.id !== 'none'
                              ? `${activeChildName}の目安: ${currentFruit.childPortionLabel}`
                              : '果物なし'}
                          </span>
                        </div>
                        <div className="flex flex-wrap gap-1.5">
                          {DESSERT_FRUIT_OPTIONS.map((fruit) => (
                            <button
                              key={fruit.id}
                              type="button"
                              onClick={() => setSelectedFruitId(fruit.id)}
                              className={`px-3 py-1.5 rounded-xl text-xs font-bold border transition cursor-pointer ${
                                selectedFruitId === fruit.id
                                  ? 'bg-rose-500 text-white border-rose-500 shadow-2xs'
                                  : 'bg-white text-stone-700 border-stone-200 hover:border-rose-300'
                              }`}
                            >
                              {fruit.label}
                            </button>
                          ))}
                        </div>
                      </div>
                    </div>

                    {/* 足りてる栄養・足りない栄養の結論サマリー */}
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                      <div className="rounded-2xl bg-emerald-50/80 border border-emerald-200/80 p-3.5">
                        <div className="flex items-center gap-1.5 text-xs font-extrabold text-emerald-800 mb-1.5">
                          <CheckCircle2 className="w-4 h-4 text-emerald-600 shrink-0" />
                          <span>この料理でしっかり摂れる栄養</span>
                        </div>
                        {sufficientNutrients.length > 0 ? (
                          <div className="flex flex-wrap gap-1.5">
                            {sufficientNutrients.map((n) => (
                              <span
                                key={n.label}
                                className="text-xs font-bold px-2 py-0.5 rounded-md bg-white text-emerald-800 border border-emerald-200"
                              >
                                {n.label}（{n.percentage}%）
                              </span>
                            ))}
                          </div>
                        ) : (
                          <p className="text-xs text-emerald-700">
                            単品では控えめです。主食や副菜と組み合わせましょう。
                          </p>
                        )}
                      </div>

                      <div className="rounded-2xl bg-amber-50/90 border border-amber-200/90 p-3.5">
                        <div className="flex items-center gap-1.5 text-xs font-extrabold text-amber-900 mb-1.5">
                          <AlertTriangle className="w-4 h-4 text-amber-600 shrink-0" />
                          <span>子供基準に対して不足気味の栄養</span>
                        </div>
                        {missingNutrients.length > 0 ? (
                          <div className="flex flex-wrap gap-1.5">
                            {missingNutrients.map((n) => (
                              <span
                                key={n.label}
                                className="text-xs font-bold px-2 py-0.5 rounded-md bg-white text-amber-900 border border-amber-300"
                              >
                                {n.label}（あと{Math.max(0, Math.round((n.target - n.actual) * 10) / 10)}
                                {n.unit}）
                              </span>
                            ))}
                          </div>
                        ) : (
                          <p className="text-xs font-bold text-emerald-700">
                            すべての栄養素が目安をクリアしています！
                          </p>
                        )}
                      </div>
                    </div>

                    {/* 栄養素ごとの詳細プログレスバー */}
                    <div className="space-y-3">
                      {Object.entries(childNutr).map(([key, item]) => {
                        const barWidth = Math.min(100, item.percentage);
                        const isSalt = item.isUpperLimit;

                        let badgeText = '';
                        let badgeClass = '';
                        let barColor = '';

                        if (isSalt) {
                          if (item.status === 'high') {
                            badgeText = '塩分やや多め（薄味レバー推奨）';
                            badgeClass = 'bg-rose-100 text-rose-800';
                            barColor = 'bg-rose-500';
                          } else {
                            badgeText = `適正範囲内（濃さ${Math.round(seasoningRatio * 100)}%反映）`;
                            badgeClass = 'bg-emerald-100 text-emerald-800';
                            barColor = 'bg-emerald-500';
                          }
                        } else {
                          if (item.status === 'great') {
                            badgeText = 'バッチリ充足！';
                            badgeClass = 'bg-emerald-100 text-emerald-800';
                            barColor = 'bg-emerald-500';
                          } else if (item.status === 'ok') {
                            badgeText = 'ほぼ充足';
                            badgeClass = 'bg-sky-100 text-sky-800';
                            barColor = 'bg-sky-500';
                          } else {
                            badgeText = '不足気味';
                            badgeClass = 'bg-amber-100 text-amber-800';
                            barColor = 'bg-amber-500';
                          }
                        }

                        return (
                          <div key={key} className="space-y-1">
                            <div className="flex items-center justify-between text-xs">
                              <div className="flex items-center gap-2">
                                <span className="font-bold text-stone-800">{item.label}</span>
                                <span
                                  className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${badgeClass}`}
                                >
                                  {badgeText}
                                </span>
                              </div>
                              <div className="text-stone-600 font-medium">
                                <strong className="text-stone-900 text-sm">{item.actual}</strong>
                                {item.unit} /{' '}
                                <span>
                                  {isSalt ? '上限目安 ' : '必要基準 '}
                                  {item.target}
                                  {item.unit}（{item.percentage}%）
                                </span>
                              </div>
                            </div>

                            <div className="w-full h-2.5 bg-stone-100 rounded-full overflow-hidden">
                              <div
                                className={`h-full rounded-full transition-all duration-300 ${barColor}`}
                                style={{ width: `${barWidth}%` }}
                              />
                            </div>
                          </div>
                        );
                      })}
                    </div>

                    {/* 管理栄養士AIからのプラス1品アドバイス */}
                    <div className="rounded-2xl bg-gradient-to-r from-orange-50 to-amber-50 border border-orange-200/80 p-4">
                      <div className="text-xs font-extrabold text-orange-900 mb-1 flex items-center gap-1.5">
                        <Sparkles className="w-4 h-4 text-orange-500 shrink-0" />
                        <span>不足栄養を補う「プラス1品」アドバイス</span>
                      </div>
                      <p className="text-xs sm:text-sm text-stone-700 leading-relaxed">
                        {analyzedRecipe.childNutritionAdvice}
                      </p>
                    </div>
                  </div>

                  {/* 右5カラム：このレシピのわが家仕様（3.5人前＆薄味）分量＆即買い物リスト */}
                  <div className="lg:col-span-5 bg-white rounded-3xl border border-stone-200/90 p-5 sm:p-6 shadow-2xs flex flex-col justify-between space-y-4">
                    <div>
                      <div className="flex items-center justify-between gap-2 mb-3">
                        <div>
                          <span className="text-xs font-bold text-orange-600">
                            元 {analyzedRecipe.servings}人前 → わが家 {totalFamilyServings}人前換算
                          </span>
                          <h3 className="text-base font-extrabold text-stone-800">
                            わが家の買い物＆調理分量
                          </h3>
                        </div>
                        <button
                          onClick={() => setInspectRecipeId(analyzedRecipe.id)}
                          className="text-xs font-bold text-orange-700 bg-orange-50 hover:bg-orange-100 px-3 py-1.5 rounded-xl transition cursor-pointer"
                        >
                          個別微調整
                        </button>
                      </div>

                      <div className="divide-y divide-stone-100 border-t border-b border-stone-100">
                        {analyzedRecipe.ingredients.map((ing) => {
                          const scaled = calculateScaledIngredient(analyzedRecipe, ing);
                          return (
                            <div
                              key={ing.id}
                              className="py-2.5 flex items-center justify-between gap-2 text-sm"
                            >
                              <div className="flex items-center gap-2 min-w-0">
                                <span className="font-bold text-stone-800 truncate">
                                  {ing.name}
                                </span>
                                {ing.isSeasoning && (
                                  <span className="text-[10px] font-bold px-1.5 py-0.5 rounded bg-emerald-100 text-emerald-800 shrink-0">
                                    濃さ{Math.round(seasoningRatio * 100)}%
                                  </span>
                                )}
                              </div>
                              <div className="shrink-0 text-right">
                                <span className="text-base font-extrabold text-orange-600">
                                  {scaled.formattedAmount}
                                </span>
                                <span className="text-xs font-bold text-stone-600 ml-0.5">
                                  {ing.unit}
                                </span>
                              </div>
                            </div>
                          );
                        })}
                      </div>
                    </div>

                    <div className="space-y-2.5 pt-2">
                      <button
                        onClick={() => {
                          let text = `🍳【${analyzedRecipe.title}】\n`;
                          text += `わが家分量（計${totalFamilyServings}人前 / 調味料${Math.round(
                            seasoningRatio * 100
                          )}%）\n\n`;
                          analyzedRecipe.ingredients.forEach((ing) => {
                            const scaled = calculateScaledIngredient(analyzedRecipe, ing);
                            text += `・${ing.name}：${scaled.formattedAmount}${ing.unit}\n`;
                          });
                          navigator.clipboard?.writeText(text);
                          showToast('このレシピのわが家分量をコピーしました！');
                        }}
                        className="w-full py-2.5 rounded-xl bg-stone-100 hover:bg-stone-200 text-stone-800 font-bold text-xs flex items-center justify-center gap-1.5 transition cursor-pointer"
                      >
                        <Copy className="w-3.5 h-3.5" />
                        <span>このレシピの買い物分量をコピー</span>
                      </button>

                      <button
                        onClick={() => {
                          handleAddRecipeToSlot('mon', 'dinner', analyzedRecipe.id);
                          setActiveTab('calendar');
                        }}
                        className="w-full py-3 rounded-xl bg-emerald-600 hover:bg-emerald-700 text-white font-bold text-xs sm:text-sm flex items-center justify-center gap-1.5 shadow-xs transition cursor-pointer"
                      >
                        <Calendar className="w-4 h-4" />
                        <span>1週間の献立表にも追加する</span>
                      </button>
                    </div>
                  </div>
                </div>
              );
            })()}
          </div>
        )}

        {/* ====================================================================
            タブ1: 1週間の献立カレンダー（月〜日 × 朝・昼・晩）
           ==================================================================== */}
        {activeTab === 'calendar' && (
          <div className="space-y-4">
            {/* 朝ごはん一括セット クイックバー */}
            <div className="bg-white rounded-2xl p-3.5 border border-stone-200/80 shadow-2xs flex flex-wrap items-center justify-between gap-3">
              <div className="flex items-center gap-2 text-xs sm:text-sm font-semibold text-stone-700">
                <Coffee className="w-4 h-4 text-amber-500 shrink-0" />
                <span>朝ごはんの定番をワンタップ一括配置：</span>
              </div>
              <div className="flex flex-wrap items-center gap-2">
                {recipes
                  .filter((r) => r.type === 'preset')
                  .slice(0, 3)
                  .map((preset) => (
                    <div
                      key={preset.id}
                      className="flex items-center bg-amber-50 border border-amber-200/80 rounded-xl overflow-hidden text-xs"
                    >
                      <span className="px-2.5 py-1.5 font-medium text-amber-900 truncate max-w-[140px]">
                        {preset.title.replace('定番の', '')}
                      </span>
                      <button
                        onClick={() => handleBulkFillBreakfast(preset.id, 'weekdays')}
                        className="px-2 py-1.5 bg-amber-200/70 hover:bg-amber-300/80 text-amber-950 font-bold border-l border-amber-200 transition cursor-pointer"
                        title="月〜金の朝にセット"
                      >
                        平日朝
                      </button>
                      <button
                        onClick={() => handleBulkFillBreakfast(preset.id, 'all')}
                        className="px-2 py-1.5 bg-amber-500 hover:bg-amber-600 text-white font-bold transition cursor-pointer"
                        title="月〜日の全朝にセット"
                      >
                        毎日朝
                      </button>
                    </div>
                  ))}
              </div>
            </div>

            {/* 週間献立グリッド（モバイル: 1列カード / デスクトップ: 2〜4列見やすいカード） */}
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
              {DAYS.map((day, dayIndex) => {
                const isFirstTrip = dayIndex <= splitDayIndex;
                // その日の全食事（朝・昼・晩）を合算した子供の1日栄養達成度を算出
                const dayRecipeIds = [
                  ...(weeklyPlan[day.key]?.breakfast || []),
                  ...(weeklyPlan[day.key]?.lunch || []),
                  ...(weeklyPlan[day.key]?.dinner || []),
                ];
                const dayRecipes = dayRecipeIds
                  .map((id) => recipes.find((r) => r.id === id))
                  .filter(Boolean);

                const dayLowNutrients = [];
                if (dayRecipes.length > 0) {
                  const totals = { calories: 0, protein: 0, calcium: 0, iron: 0, fiber: 0, salt: 0 };
                  dayRecipes.forEach((rec) => {
                    const mealNutr = calculateChildNutritionForRecipe(rec, 'daily', {
                      stapleId: 'rice-white',
                      withHarvardSoup: false,
                      childRatio: (joRatio + soRatio) / 2,
                    });
                    Object.keys(totals).forEach((k) => {
                      totals[k] += mealNutr[k].actual;
                    });
                  });
                  Object.entries(CHILD_DAILY_NUTRITION_TARGETS).forEach(([k, meta]) => {
                    if (!meta.isUpperLimit && totals[k] < meta.daily * 0.75) {
                      dayLowNutrients.push(meta.label);
                    }
                  });
                }

                return (
                  <div
                    key={day.key}
                    className="bg-white rounded-2xl border border-stone-200/90 shadow-2xs overflow-hidden flex flex-col"
                  >
                    {/* 曜日ヘッダー（週2回買い出しのどちらのグループかバッジ表示） */}
                    <div
                      className={`px-4 py-2.5 border-b flex items-center justify-between ${
                        day.key === 'sat'
                          ? 'bg-sky-50/70 border-sky-100'
                          : day.key === 'sun'
                          ? 'bg-rose-50/70 border-rose-100'
                          : 'bg-stone-50 border-stone-100'
                      }`}
                    >
                      <div className="flex items-center gap-2">
                        <span
                          className={`w-7 h-7 rounded-lg flex items-center justify-center text-sm font-bold ${
                            day.key === 'sat'
                              ? 'bg-sky-500 text-white'
                              : day.key === 'sun'
                              ? 'bg-rose-500 text-white'
                              : 'bg-stone-800 text-white'
                          }`}
                        >
                          {day.label}
                        </span>
                        <span className="font-bold text-sm text-stone-700">{day.fullLabel}</span>
                      </div>

                      <span
                        className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${
                          isFirstTrip
                            ? 'bg-emerald-100 text-emerald-800'
                            : 'bg-indigo-100 text-indigo-800'
                        }`}
                      >
                        {isFirstTrip ? '第1回買出し' : '第2回買出し'}
                      </span>
                    </div>

                    {/* その日の子供栄養バランスミニバー */}
                    {dayRecipes.length > 0 && (
                      <div className="px-3 py-1.5 bg-amber-50/70 border-b border-amber-100 flex items-center justify-between gap-1 text-[10px]">
                        <span className="font-bold text-stone-600 shrink-0">子供栄養:</span>
                        {dayLowNutrients.length === 0 ? (
                          <span className="font-bold text-emerald-700">1日目安バッチリ！</span>
                        ) : (
                          <span className="font-bold text-amber-800 truncate">
                            不足気味: {dayLowNutrients.slice(0, 2).join('・')}
                            {dayLowNutrients.length > 2 ? '他' : ''}
                          </span>
                        )}
                      </div>
                    )}

                    {/* 朝・昼・晩 スロット */}
                    <div className="p-3 space-y-2.5 flex-1">
                      {MEAL_TYPES.map((meal) => {
                        const MealIcon = meal.icon;
                        const assignedIds = weeklyPlan[day.key]?.[meal.key] || [];

                        return (
                          <div
                            key={meal.key}
                            className="rounded-xl bg-stone-50/80 border border-stone-200/60 p-2.5"
                          >
                            <div className="flex items-center justify-between mb-1.5">
                              <div className="flex items-center gap-1.5">
                                <span
                                  className={`w-5 h-5 rounded-md ${meal.bg} ${meal.color} flex items-center justify-center`}
                                >
                                  <MealIcon className="w-3.5 h-3.5" />
                                </span>
                                <span className="text-xs font-bold text-stone-600">
                                  {meal.label}ごはん
                                </span>
                              </div>

                              <button
                                onClick={() =>
                                  setSelectedSlot({ dayKey: day.key, mealKey: meal.key })
                                }
                                className="inline-flex items-center gap-1 text-xs font-bold text-orange-600 hover:text-orange-700 bg-orange-50 hover:bg-orange-100 px-2 py-1 rounded-lg transition cursor-pointer"
                              >
                                <Plus className="w-3.5 h-3.5" />
                                <span>追加</span>
                              </button>
                            </div>

                            {/* 登録済みメニューリスト */}
                            {assignedIds.length === 0 ? (
                              <div className="text-[11px] text-stone-400 py-1 pl-1">
                                未登録
                              </div>
                            ) : (
                              <div className="space-y-1.5">
                                {assignedIds.map((rId, idx) => {
                                  const recipe = recipes.find((r) => r.id === rId);
                                  if (!recipe) return null;
                                  return (
                                    <div
                                      key={`${rId}-${idx}`}
                                      className="bg-white border border-stone-200/90 rounded-lg p-2 flex items-center justify-between gap-2 shadow-2xs group"
                                    >
                                      <button
                                        onClick={() => setInspectRecipeId(recipe.id)}
                                        className="text-left flex-1 min-w-0 cursor-pointer"
                                        title="タップしてわが家仕様の分量・調味料を確認・微調整"
                                      >
                                        <div className="text-xs font-bold text-stone-800 truncate group-hover:text-orange-600 transition">
                                          {recipe.title}
                                        </div>
                                        <div className="text-[10px] text-stone-500 flex items-center gap-1.5 mt-0.5">
                                          <span>
                                            元{recipe.servings}人前 →{' '}
                                            <strong className="text-orange-600">
                                              {totalFamilyServings}人前換算
                                            </strong>
                                          </span>
                                        </div>
                                      </button>

                                      <button
                                        onClick={() =>
                                          handleRemoveRecipeFromSlot(day.key, meal.key, idx)
                                        }
                                        className="p-1 text-stone-400 hover:text-rose-500 rounded-md hover:bg-rose-50 transition shrink-0 cursor-pointer"
                                        aria-label="献立から削除"
                                      >
                                        <X className="w-3.5 h-3.5" />
                                      </button>
                                    </div>
                                  );
                                })}
                              </div>
                            )}
                          </div>
                        );
                      })}
                    </div>
                  </div>
                );
              })}
            </div>

            {/* 下部アクション：買い物リストへ進む */}
            <div className="pt-2 flex justify-center">
              <button
                onClick={() => setActiveTab('shopping')}
                className="w-full sm:w-auto px-8 py-3.5 rounded-2xl bg-emerald-600 hover:bg-emerald-700 text-white font-bold text-sm sm:text-base shadow-md flex items-center justify-center gap-2.5 transition cursor-pointer"
              >
                <ShoppingCart className="w-5 h-5" />
                <span>この献立で週2回の買い物リストを見る</span>
                <ChevronRight className="w-5 h-5" />
              </button>
            </div>
          </div>
        )}

        {/* ====================================================================
            タブ2: レシピ動画解析 ＆ 定番メニュー個別登録
           ==================================================================== */}
        {activeTab === 'recipes' && (
          <div className="grid grid-cols-1 lg:grid-cols-12 gap-6">
            {/* 左カラム：新規レシピ・定番登録フォーム */}
            <div className="lg:col-span-5 space-y-4">
              <div className="bg-white rounded-2xl border border-stone-200/90 p-5 shadow-2xs">
                <h2 className="text-base font-bold text-stone-800 mb-3 flex items-center gap-2">
                  <Plus className="w-5 h-5 text-orange-500" />
                  <span>新しい料理・定番セットを登録</span>
                </h2>

                {/* 登録方法切り替えタブ */}
                <div className="grid grid-cols-2 gap-1.5 bg-stone-100 p-1 rounded-xl mb-4">
                  <button
                    type="button"
                    onClick={() => setRecipeFormMode('ai')}
                    className={`py-2 px-3 rounded-lg text-xs font-bold flex items-center justify-center gap-1.5 transition cursor-pointer ${
                      recipeFormMode === 'ai'
                        ? 'bg-white text-orange-600 shadow-2xs'
                        : 'text-stone-600 hover:text-stone-800'
                    }`}
                  >
                    <Sparkles className="w-3.5 h-3.5" />
                    <span>動画URL・テキストAI解析</span>
                  </button>
                  <button
                    type="button"
                    onClick={() => setRecipeFormMode('manual')}
                    className={`py-2 px-3 rounded-lg text-xs font-bold flex items-center justify-center gap-1.5 transition cursor-pointer ${
                      recipeFormMode === 'manual'
                        ? 'bg-white text-orange-600 shadow-2xs'
                        : 'text-stone-600 hover:text-stone-800'
                    }`}
                  >
                    <Edit3 className="w-3.5 h-3.5" />
                    <span>個別・定番フォーマット</span>
                  </button>
                </div>

                {/* AI解析フォーム */}
                {recipeFormMode === 'ai' && (
                  <div className="space-y-3">
                    <div className="bg-orange-50/70 border border-orange-200/80 rounded-xl p-3 text-xs text-stone-600 leading-relaxed">
                      <p className="font-bold text-orange-800 mb-1 flex items-center gap-1">
                        <Video className="w-3.5 h-3.5" />
                        レシピ動画のURLや概要欄を貼るだけ！
                      </p>
                      AIが「何人前のレシピか」「食材」「調味料」を自動で読み取り、大人2人＋子供2人（{totalFamilyServings}人前）＆薄味設定に合わせて変換します。
                    </div>

                    <textarea
                      rows={5}
                      value={aiInputText}
                      onChange={(e) => setAiInputText(e.target.value)}
                      placeholder="例：&#10;https://www.youtube.com/watch?v=...&#10;または料理名・概要欄の材料リストを貼り付け&#10;（例：豚の生姜焼き 2人前 豚ロース300g、玉ねぎ1個、醤油大さじ2、みりん大さじ2、生姜1片）"
                      className="w-full rounded-xl border border-stone-300 p-3 text-sm focus:outline-none focus:ring-2 focus:ring-orange-500/40 focus:border-orange-500"
                    />

                    <button
                      type="button"
                      onClick={handleAiExtract}
                      disabled={isAnalyzing}
                      className="w-full py-3 rounded-xl bg-gradient-to-r from-orange-500 to-amber-500 hover:from-orange-600 hover:to-amber-600 disabled:opacity-60 text-white font-bold text-sm shadow-sm flex items-center justify-center gap-2 transition cursor-pointer"
                    >
                      {isAnalyzing ? (
                        <>
                          <RefreshCw className="w-4 h-4 animate-spin" />
                          <span>AIがレシピと何人前かを解析中...</span>
                        </>
                      ) : (
                        <>
                          <Sparkles className="w-4 h-4" />
                          <span>AIで材料と人前を自動抽出する</span>
                        </>
                      )}
                    </button>
                  </div>
                )}

                {/* 個別・定番・編集確認フォーム */}
                {recipeFormMode === 'manual' && (
                  <form onSubmit={handleSaveNewRecipe} className="space-y-3.5">
                    <div>
                      <label className="block text-xs font-bold text-stone-700 mb-1">
                        種別（朝ごはん定番 or レシピ）
                      </label>
                      <div className="grid grid-cols-2 gap-2">
                        <button
                          type="button"
                          onClick={() => {
                            setManualType('preset');
                            setManualServings(totalFamilyServings);
                          }}
                          className={`py-2 px-3 rounded-xl border text-xs font-bold transition cursor-pointer ${
                            manualType === 'preset'
                              ? 'bg-amber-50 border-amber-400 text-amber-900'
                              : 'bg-white border-stone-200 text-stone-600'
                          }`}
                        >
                          朝ごはん・定番セット
                        </button>
                        <button
                          type="button"
                          onClick={() => {
                            setManualType('video');
                            setManualServings(2);
                          }}
                          className={`py-2 px-3 rounded-xl border text-xs font-bold transition cursor-pointer ${
                            manualType === 'video'
                              ? 'bg-orange-50 border-orange-400 text-orange-900'
                              : 'bg-white border-stone-200 text-stone-600'
                          }`}
                        >
                          動画・料理レシピ
                        </button>
                      </div>
                    </div>

                    <div className="grid grid-cols-1 sm:grid-cols-3 gap-2.5">
                      <div className="sm:col-span-2">
                        <label className="block text-xs font-bold text-stone-700 mb-1">
                          料理名・セット名 *
                        </label>
                        <input
                          type="text"
                          value={manualTitle}
                          onChange={(e) => setManualTitle(e.target.value)}
                          placeholder="例：毎朝のパン＆フルーツセット"
                          className="w-full rounded-xl border border-stone-300 px-3 py-2 text-sm focus:outline-none focus:border-orange-500"
                          required
                        />
                      </div>

                      <div>
                        <label className="block text-xs font-bold text-stone-700 mb-1">
                          基準人数（何人前）
                        </label>
                        <input
                          type="number"
                          step="0.5"
                          min="0.5"
                          max="20"
                          value={manualServings}
                          onChange={(e) => setManualServings(e.target.value)}
                          className="w-full rounded-xl border border-stone-300 px-3 py-2 text-sm font-bold text-center focus:outline-none focus:border-orange-500"
                        />
                      </div>
                    </div>

                    {manualType === 'video' && (
                      <div>
                        <label className="block text-xs font-bold text-stone-700 mb-1">
                          動画URL（メモ用・任意）
                        </label>
                        <input
                          type="url"
                          value={manualUrl}
                          onChange={(e) => setManualUrl(e.target.value)}
                          placeholder="https://youtube.com/..."
                          className="w-full rounded-xl border border-stone-300 px-3 py-1.5 text-xs focus:outline-none focus:border-orange-500"
                        />
                      </div>
                    )}

                    {/* 材料リスト入力 */}
                    <div>
                      <div className="flex items-center justify-between mb-1.5">
                        <label className="text-xs font-bold text-stone-700">
                          材料リスト（調味料チェックで薄味レバー連動）
                        </label>
                      </div>

                      <div className="space-y-2 max-h-[340px] overflow-y-auto pr-1">
                        {manualIngredients.map((ing, idx) => (
                          <div
                            key={ing.id}
                            className="p-2.5 rounded-xl bg-stone-50 border border-stone-200 space-y-2"
                          >
                            <div className="flex items-center gap-1.5">
                              <input
                                type="text"
                                value={ing.name}
                                onChange={(e) => {
                                  const val = e.target.value;
                                  setManualIngredients((prev) =>
                                    prev.map((item, i) => (i === idx ? { ...item, name: val } : item))
                                  );
                                }}
                                placeholder="材料名（例：食パン、醤油）"
                                className="flex-1 min-w-0 rounded-lg border border-stone-300 bg-white px-2.5 py-1.5 text-xs"
                              />
                              <input
                                type="number"
                                step="0.1"
                                min="0"
                                value={ing.amount}
                                onChange={(e) => {
                                  const val = e.target.value;
                                  setManualIngredients((prev) =>
                                    prev.map((item, i) =>
                                      i === idx ? { ...item, amount: val } : item
                                    )
                                  );
                                }}
                                className="w-16 rounded-lg border border-stone-300 bg-white px-2 py-1.5 text-xs text-right font-bold"
                              />
                              <input
                                type="text"
                                value={ing.unit}
                                onChange={(e) => {
                                  const val = e.target.value;
                                  setManualIngredients((prev) =>
                                    prev.map((item, i) => (i === idx ? { ...item, unit: val } : item))
                                  );
                                }}
                                placeholder="単位"
                                className="w-14 rounded-lg border border-stone-300 bg-white px-2 py-1.5 text-xs text-center"
                              />
                              {manualIngredients.length > 1 && (
                                <button
                                  type="button"
                                  onClick={() =>
                                    setManualIngredients((prev) =>
                                      prev.filter((_, i) => i !== idx)
                                    )
                                  }
                                  className="p-1 text-stone-400 hover:text-rose-500 cursor-pointer"
                                >
                                  <Trash2 className="w-3.5 h-3.5" />
                                </button>
                              )}
                            </div>

                            <div className="flex items-center justify-between gap-2 text-[11px]">
                              <select
                                value={ing.category}
                                onChange={(e) => {
                                  const val = e.target.value;
                                  setManualIngredients((prev) =>
                                    prev.map((item, i) =>
                                      i === idx ? { ...item, category: val } : item
                                    )
                                  );
                                }}
                                className="rounded-lg border border-stone-200 bg-white px-2 py-1 text-stone-600"
                              >
                                {INGREDIENT_CATEGORIES.map((c) => (
                                  <option key={c} value={c}>
                                    {c}
                                  </option>
                                ))}
                              </select>

                              <label className="inline-flex items-center gap-1.5 cursor-pointer select-none font-medium text-stone-700">
                                <input
                                  type="checkbox"
                                  checked={ing.isSeasoning}
                                  onChange={(e) => {
                                    const checked = e.target.checked;
                                    setManualIngredients((prev) =>
                                      prev.map((item, i) =>
                                        i === idx
                                          ? {
                                              ...item,
                                              isSeasoning: checked,
                                              category: checked ? '調味料・油' : item.category,
                                            }
                                          : item
                                      )
                                    );
                                  }}
                                  className="rounded text-orange-500 focus:ring-orange-500"
                                />
                                <span>調味料（薄味調整対象）</span>
                              </label>
                            </div>
                          </div>
                        ))}
                      </div>

                      <button
                        type="button"
                        onClick={() =>
                          setManualIngredients((prev) => [
                            ...prev,
                            {
                              id: `m-${Date.now()}`,
                              name: '',
                              amount: 1,
                              unit: '個',
                              category: '野菜・果物',
                              isSeasoning: false,
                              customMultiplier: 1,
                            },
                          ])
                        }
                        className="mt-2 w-full py-2 rounded-xl border border-dashed border-stone-300 hover:border-orange-400 text-xs font-bold text-stone-600 hover:text-orange-600 flex items-center justify-center gap-1 transition cursor-pointer"
                      >
                        <Plus className="w-3.5 h-3.5" />
                        <span>材料行を追加</span>
                      </button>
                    </div>

                    <button
                      type="submit"
                      className="w-full py-3 rounded-xl bg-stone-800 hover:bg-stone-900 text-white font-bold text-sm shadow-sm transition cursor-pointer"
                    >
                      マイレシピ・定番リストに保存する
                    </button>
                  </form>
                )}
              </div>
            </div>

            {/* 右カラム：登録済みレシピ＆定番セット一覧（わが家分量プレビュー＆個別微調整） */}
            <div className="lg:col-span-7 space-y-3">
              <div className="flex items-center justify-between">
                <h2 className="text-base font-bold text-stone-800">
                  登録済みのレシピ＆定番セット（{recipes.length}件）
                </h2>
                <span className="text-xs text-stone-500">
                  カードをタップして調味料の個別微調整ができます
                </span>
              </div>

              <div className="grid grid-cols-1 md:grid-cols-2 gap-3.5">
                {recipes.map((recipe) => {
                  const scaleRatio =
                    Math.round((totalFamilyServings / (recipe.servings || 1)) * 100) / 100;
                  return (
                    <div
                      key={recipe.id}
                      className="bg-white rounded-2xl border border-stone-200/90 p-4 shadow-2xs hover:border-orange-300 transition flex flex-col justify-between"
                    >
                      <div>
                        <div className="flex items-start justify-between gap-2 mb-2">
                          <div className="flex flex-wrap items-center gap-1.5">
                            <span
                              className={`text-[10px] font-bold px-2 py-0.5 rounded-md ${
                                recipe.type === 'video'
                                  ? 'bg-orange-100 text-orange-800'
                                  : 'bg-amber-100 text-amber-800'
                              }`}
                            >
                              {recipe.type === 'video' ? '動画レシピ' : '定番・個別'}
                            </span>
                            <span className="text-[11px] text-stone-500 font-medium">
                              元{recipe.servings}人前 →{' '}
                              <strong className="text-orange-600">{scaleRatio}倍</strong>
                            </span>
                          </div>

                          <button
                            onClick={() => handleDeleteRecipe(recipe.id)}
                            className="p-1 text-stone-400 hover:text-rose-500 rounded-lg hover:bg-rose-50 transition cursor-pointer"
                            title="このレシピを削除"
                          >
                            <Trash2 className="w-4 h-4" />
                          </button>
                        </div>

                        <h3 className="font-bold text-sm text-stone-800 mb-2.5 leading-snug">
                          {recipe.title}
                        </h3>

                        {/* わが家仕様に換算済みの材料プレビュー */}
                        <div className="bg-stone-50 rounded-xl p-2.5 space-y-1 mb-3">
                          {recipe.ingredients.slice(0, 5).map((ing) => {
                            const scaled = calculateScaledIngredient(recipe, ing);
                            return (
                              <div
                                key={ing.id}
                                className="flex items-center justify-between text-xs"
                              >
                                <span className="text-stone-700 truncate pr-2 flex items-center gap-1">
                                  {ing.name}
                                  {ing.isSeasoning && (
                                    <span className="text-[9px] px-1 py-0.2 bg-emerald-100 text-emerald-800 rounded font-bold">
                                      味
                                    </span>
                                  )}
                                </span>
                                <span className="font-bold text-stone-800 shrink-0">
                                  {scaled.formattedAmount}
                                  <span className="text-[11px] font-normal text-stone-500 ml-0.5">
                                    {ing.unit}
                                  </span>
                                </span>
                              </div>
                            );
                          })}
                          {recipe.ingredients.length > 5 && (
                            <div className="text-[11px] text-stone-400 text-right pt-0.5">
                              ほか {recipe.ingredients.length - 5} 品目...
                            </div>
                          )}
                        </div>
                      </div>

                      <button
                        onClick={() => setInspectRecipeId(recipe.id)}
                        className="w-full py-2 rounded-xl bg-orange-50 hover:bg-orange-100 text-orange-700 font-bold text-xs flex items-center justify-center gap-1.5 transition cursor-pointer"
                      >
                        <Sliders className="w-3.5 h-3.5" />
                        <span>全材料・調味料の個別微調整</span>
                      </button>
                    </div>
                  );
                })}
              </div>
            </div>
          </div>
        )}

        {/* ====================================================================
            タブ3: 週2回のまとめ買いリスト（一括合算・売り場別）
           ==================================================================== */}
        {activeTab === 'shopping' && (
          <div className="space-y-5">
            {/* 週2回の買い出し区切り設定パネル */}
            <div className="bg-white rounded-2xl border border-stone-200/90 p-4 shadow-2xs space-y-4">
              <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-3">
                <div>
                  <h2 className="text-base font-bold text-stone-800 flex items-center gap-2">
                    <ShoppingCart className="w-5 h-5 text-emerald-600" />
                    <span>週2回の買い出しスケジュール設定</span>
                  </h2>
                  <p className="text-xs text-stone-500 mt-0.5">
                    1週間の献立を2回の買い物に自動分割し、同じ食材・調味料を合算します
                  </p>
                </div>

                <div className="flex items-center gap-2">
                  <label className="text-xs font-bold text-stone-600 shrink-0">
                    第1回の買い出し範囲:
                  </label>
                  <select
                    value={splitDayIndex}
                    onChange={(e) => setSplitDayIndex(Number(e.target.value))}
                    className="rounded-xl border border-stone-300 bg-stone-50 px-3 py-1.5 text-xs sm:text-sm font-bold text-stone-800 focus:outline-none focus:border-emerald-500"
                  >
                    <option value={2}>月〜水（3日分） / 木〜日（4日分）</option>
                    <option value={3}>月〜木（4日分） / 金〜日（3日分）</option>
                    <option value={4}>月〜金（5日分） / 土〜日（2日分）</option>
                  </select>
                </div>
              </div>

              {/* 第1回・第2回 切り替えカード */}
              <div className="grid grid-cols-2 gap-3">
                <button
                  onClick={() => setActiveShoppingTrip(1)}
                  className={`p-3.5 rounded-2xl border-2 text-left transition cursor-pointer ${
                    activeShoppingTrip === 1
                      ? 'border-emerald-500 bg-emerald-50/70'
                      : 'border-stone-200 bg-white hover:border-stone-300'
                  }`}
                >
                  <div className="flex items-center justify-between mb-1">
                    <span className="text-xs font-bold text-emerald-700 bg-emerald-100 px-2 py-0.5 rounded-md">
                      第1回 買い出し
                    </span>
                    <span className="text-xs font-bold text-stone-500">
                      {shoppingLists.trip1.items.length}品目
                    </span>
                  </div>
                  <div className="text-sm sm:text-base font-bold text-stone-800">
                    {shoppingLists.trip1.label}
                  </div>
                </button>

                <button
                  onClick={() => setActiveShoppingTrip(2)}
                  className={`p-3.5 rounded-2xl border-2 text-left transition cursor-pointer ${
                    activeShoppingTrip === 2
                      ? 'border-indigo-500 bg-indigo-50/70'
                      : 'border-stone-200 bg-white hover:border-stone-300'
                  }`}
                >
                  <div className="flex items-center justify-between mb-1">
                    <span className="text-xs font-bold text-indigo-700 bg-indigo-100 px-2 py-0.5 rounded-md">
                      第2回 買い出し
                    </span>
                    <span className="text-xs font-bold text-stone-500">
                      {shoppingLists.trip2.items.length}品目
                    </span>
                  </div>
                  <div className="text-sm sm:text-base font-bold text-stone-800">
                    {shoppingLists.trip2.label}
                  </div>
                </button>
              </div>

              {/* アクションバー（調味料除外トグル＆LINE等へのコピーボタン） */}
              <div className="flex flex-wrap items-center justify-between gap-3 pt-2 border-t border-stone-100">
                <label className="inline-flex items-center gap-2 text-xs font-medium text-stone-700 cursor-pointer select-none">
                  <input
                    type="checkbox"
                    checked={excludePantryStaples}
                    onChange={(e) => setExcludePantryStaples(e.target.checked)}
                    className="rounded text-emerald-600 focus:ring-emerald-500"
                  />
                  <span>家にある基本調味料（醤油・みりん等）はリストから隠す</span>
                </label>

                <button
                  onClick={handleCopyShoppingList}
                  className="inline-flex items-center gap-1.5 px-4 py-2 rounded-xl bg-stone-800 hover:bg-stone-900 text-white text-xs font-bold transition cursor-pointer"
                >
                  <Copy className="w-3.5 h-3.5" />
                  <span>リストをテキストコピー</span>
                </button>
              </div>
            </div>

            {/* 売り場別・合算買い物リスト表示 */}
            {(() => {
              const currentTrip =
                activeShoppingTrip === 1 ? shoppingLists.trip1 : shoppingLists.trip2;
              const visibleItems = currentTrip.items.filter(
                (item) => !(excludePantryStaples && item.isSeasoning)
              );

              if (visibleItems.length === 0) {
                return (
                  <div className="bg-white rounded-2xl border border-stone-200 p-10 text-center space-y-2">
                    <p className="text-sm font-bold text-stone-700">
                      この期間の買い物リストはまだ空です
                    </p>
                    <p className="text-xs text-stone-500">
                      「1週間の献立表」から朝・昼・晩の料理を追加すると自動で計算されます
                    </p>
                  </div>
                );
              }

              return (
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {INGREDIENT_CATEGORIES.map((category) => {
                    const catItems = visibleItems.filter((i) => i.category === category);
                    if (catItems.length === 0) return null;

                    return (
                      <div
                        key={category}
                        className="bg-white rounded-2xl border border-stone-200/90 overflow-hidden shadow-2xs"
                      >
                        <div className="bg-stone-50 px-4 py-2.5 border-b border-stone-200/70 flex items-center justify-between">
                          <h3 className="font-bold text-sm text-stone-800">{category}</h3>
                          <span className="text-xs font-bold text-stone-500">
                            {catItems.length}点
                          </span>
                        </div>

                        <div className="divide-y divide-stone-100">
                          {catItems.map((item) => {
                            const checkKey = `${activeShoppingTrip}-${item.key}`;
                            const isChecked = Boolean(checkedItems[checkKey]);

                            return (
                              <div
                                key={item.key}
                                onClick={() =>
                                  setCheckedItems((prev) => ({
                                    ...prev,
                                    [checkKey]: !prev[checkKey],
                                  }))
                                }
                                className={`px-4 py-3 flex items-center justify-between gap-3 cursor-pointer transition ${
                                  isChecked ? 'bg-stone-50/80 opacity-55' : 'hover:bg-emerald-50/30'
                                }`}
                              >
                                <div className="flex items-center gap-3 min-w-0">
                                  <div
                                    className={`w-5 h-5 rounded-md border flex items-center justify-center shrink-0 transition ${
                                      isChecked
                                        ? 'bg-emerald-600 border-emerald-600 text-white'
                                        : 'border-stone-300 bg-white'
                                    }`}
                                  >
                                    {isChecked && <Check className="w-3.5 h-3.5" />}
                                  </div>

                                  <div className="min-w-0">
                                    <div
                                      className={`text-sm font-bold truncate ${
                                        isChecked ? 'line-through text-stone-400' : 'text-stone-800'
                                      }`}
                                    >
                                      {item.name}
                                    </div>
                                    <div className="text-[11px] text-stone-400 truncate">
                                      使用: {item.usedIn.join(' / ')}
                                    </div>
                                  </div>
                                </div>

                                <div className="text-right shrink-0">
                                  <span
                                    className={`text-base font-extrabold ${
                                      isChecked
                                        ? 'text-stone-400'
                                        : item.isSeasoning
                                        ? 'text-orange-600'
                                        : 'text-emerald-700'
                                    }`}
                                  >
                                    {item.formattedAmount}
                                  </span>
                                  <span className="text-xs font-bold text-stone-600 ml-0.5">
                                    {item.unit}
                                  </span>
                                  {item.isSeasoning && (
                                    <div className="text-[10px] text-orange-500 font-medium">
                                      濃さ{Math.round(seasoningRatio * 100)}%反映
                                    </div>
                                  )}
                                </div>
                              </div>
                            );
                          })}
                        </div>
                      </div>
                    );
                  })}
                </div>
              );
            })()}
          </div>
        )}
      </main>

      {/* ======================================================================
          モーダル1: 献立スロットへのレシピ選択モーダル
         ====================================================================== */}
      {selectedSlot && (
        <div className="fixed inset-0 z-50 bg-stone-900/50 backdrop-blur-xs flex items-end sm:items-center justify-center p-0 sm:p-4">
          <div className="bg-white w-full sm:max-w-lg rounded-t-3xl sm:rounded-3xl shadow-2xl overflow-hidden max-h-[85vh] flex flex-col">
            <div className="px-5 py-4 border-b border-stone-200 flex items-center justify-between">
              <div>
                <h3 className="font-bold text-base text-stone-800">
                  {DAYS.find((d) => d.key === selectedSlot.dayKey)?.fullLabel}の
                  {MEAL_TYPES.find((m) => m.key === selectedSlot.mealKey)?.label}ごはんを追加
                </h3>
                <p className="text-xs text-stone-500">
                  登録済みの動画レシピ・定番セットから選択してください
                </p>
              </div>
              <button
                onClick={() => setSelectedSlot(null)}
                className="p-2 rounded-full hover:bg-stone-100 text-stone-500 cursor-pointer"
              >
                <X className="w-5 h-5" />
              </button>
            </div>

            <div className="p-4 overflow-y-auto space-y-2.5 flex-1">
              {recipes.map((rec) => (
                <button
                  key={rec.id}
                  onClick={() =>
                    handleAddRecipeToSlot(selectedSlot.dayKey, selectedSlot.mealKey, rec.id)
                  }
                  className="w-full text-left p-3.5 rounded-2xl border border-stone-200 hover:border-orange-400 hover:bg-orange-50/50 transition flex items-center justify-between gap-3 cursor-pointer"
                >
                  <div className="min-w-0">
                    <div className="flex items-center gap-1.5 mb-1">
                      <span
                        className={`text-[10px] font-bold px-2 py-0.5 rounded ${
                          rec.type === 'video'
                            ? 'bg-orange-100 text-orange-800'
                            : 'bg-amber-100 text-amber-800'
                        }`}
                      >
                        {rec.type === 'video' ? '動画レシピ' : '定番セット'}
                      </span>
                      <span className="text-[11px] text-stone-500">
                        基準{rec.servings}人前 → {totalFamilyServings}人前換算
                      </span>
                    </div>
                    <div className="font-bold text-sm text-stone-800 truncate">{rec.title}</div>
                  </div>
                  <Plus className="w-5 h-5 text-orange-500 shrink-0" />
                </button>
              ))}
            </div>

            <div className="p-3 bg-stone-50 border-t border-stone-200 flex justify-between items-center">
              <span className="text-xs text-stone-500">新しいレシピを入れたい場合</span>
              <button
                onClick={() => {
                  setSelectedSlot(null);
                  setActiveTab('recipes');
                }}
                className="text-xs font-bold text-orange-600 hover:text-orange-700 px-3 py-1.5 rounded-lg bg-orange-100/70 cursor-pointer"
              >
                + 新規レシピ登録へ
              </button>
            </div>
          </div>
        </div>
      )}

      {/* ======================================================================
          モーダル2: わが家仕様の分量詳細 ＆ 個別調味料の微増減モーダル
         ====================================================================== */}
      {inspectedRecipe && (
        <div className="fixed inset-0 z-50 bg-stone-900/50 backdrop-blur-xs flex items-end sm:items-center justify-center p-0 sm:p-4">
          <div className="bg-white w-full sm:max-w-xl rounded-t-3xl sm:rounded-3xl shadow-2xl overflow-hidden max-h-[88vh] flex flex-col">
            <div className="px-5 py-4 border-b border-stone-200 flex items-start justify-between gap-3">
              <div>
                <div className="flex items-center gap-2 mb-1">
                  <span className="text-xs font-bold px-2 py-0.5 rounded bg-orange-100 text-orange-800">
                    元 {inspectedRecipe.servings}人前 → わが家 {totalFamilyServings}人前換算
                  </span>
                  <span className="text-xs font-bold px-2 py-0.5 rounded bg-emerald-100 text-emerald-800">
                    全体調味料 {Math.round(seasoningRatio * 100)}%
                  </span>
                </div>
                <h3 className="font-bold text-base sm:text-lg text-stone-800">
                  {inspectedRecipe.title}
                </h3>
              </div>
              <button
                onClick={() => setInspectRecipeId(null)}
                className="p-2 rounded-full hover:bg-stone-100 text-stone-500 shrink-0 cursor-pointer"
              >
                <X className="w-5 h-5" />
              </button>
            </div>

            <div className="p-4 bg-amber-50/60 border-b border-amber-200/60 flex items-center gap-2 text-xs text-stone-700">
              <Info className="w-4 h-4 text-amber-600 shrink-0" />
              <span>
                「- / +」ボタンで、この料理の特定の調味料や食材だけを個別に少し増減できます（買い物リストにも即反映されます）。
              </span>
            </div>

            <div className="p-4 overflow-y-auto space-y-2 flex-1">
              {inspectedRecipe.ingredients.map((ing) => {
                const scaled = calculateScaledIngredient(inspectedRecipe, ing);
                const customPct = Math.round((ing.customMultiplier ?? 1.0) * 100);

                return (
                  <div
                    key={ing.id}
                    className="p-3 rounded-xl border border-stone-200/90 bg-white flex items-center justify-between gap-3"
                  >
                    <div className="min-w-0">
                      <div className="flex items-center gap-1.5">
                        <span className="font-bold text-sm text-stone-800">{ing.name}</span>
                        {ing.isSeasoning && (
                          <span className="text-[10px] font-bold px-1.5 py-0.5 rounded bg-orange-100 text-orange-700">
                            調味料
                          </span>
                        )}
                      </div>
                      <div className="text-[11px] text-stone-400 mt-0.5">
                        元レシピ: {ing.amount}
                        {ing.unit}
                        {customPct !== 100 && (
                          <span className="ml-2 text-orange-600 font-bold">
                            (個別調整 {customPct}%)
                          </span>
                        )}
                      </div>
                    </div>

                    <div className="flex items-center gap-3 shrink-0">
                      {/* 個別微増減コントロール */}
                      <div className="flex items-center gap-1 bg-stone-100 rounded-lg p-1">
                        <button
                          onClick={() =>
                            handleAdjustIngredientMultiplier(inspectedRecipe.id, ing.id, -0.1)
                          }
                          className="w-6 h-6 rounded-md bg-white hover:bg-stone-200 flex items-center justify-center text-stone-700 shadow-2xs cursor-pointer"
                          title="10%減らす"
                        >
                          <Minus className="w-3 h-3" />
                        </button>
                        <span className="text-[11px] font-bold w-9 text-center text-stone-600">
                          {customPct}%
                        </span>
                        <button
                          onClick={() =>
                            handleAdjustIngredientMultiplier(inspectedRecipe.id, ing.id, 0.1)
                          }
                          className="w-6 h-6 rounded-md bg-white hover:bg-stone-200 flex items-center justify-center text-stone-700 shadow-2xs cursor-pointer"
                          title="10%増やす"
                        >
                          <Plus className="w-3 h-3" />
                        </button>
                      </div>

                      {/* 換算後の最終分量 */}
                      <div className="w-20 text-right">
                        <span className="text-base font-extrabold text-orange-600">
                          {scaled.formattedAmount}
                        </span>
                        <span className="text-xs font-bold text-stone-600 ml-0.5">{ing.unit}</span>
                      </div>
                    </div>
                  </div>
                );
              })}
            </div>

            <div className="p-4 border-t border-stone-200 flex justify-end">
              <button
                onClick={() => setInspectRecipeId(null)}
                className="px-6 py-2.5 rounded-xl bg-stone-800 hover:bg-stone-900 text-white text-sm font-bold cursor-pointer"
              >
                完了
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
