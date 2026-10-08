import React, { useState, useEffect } from 'react';
// 배포용: 실제 Firebase SDK 임포트
import { initializeApp } from 'firebase/app';
import { getFirestore, collection, addDoc, serverTimestamp } from 'firebase/firestore';
import { Loader2, Utensils, ArrowRight, Home, Plus, ChevronLeft, RefreshCw, Star, MapPin, ExternalLink } from 'lucide-react';

// 미리보기 환경(ES2015 컴파일러)에서 경고가 뜨지 않도록 범용 환경변수(process.env)만 사용합니다.
const getEnv = (key, fallback) => {
  if (typeof process !== 'undefined' && process.env && process.env[key]) return process.env[key];
  if (typeof window !== 'undefined' && window[key]) return window[key];
  return fallback;
};

// Vercel에서 설정할 Firebase 환경변수들
const firebaseConfig = {
  apiKey: getEnv('VITE_FIREBASE_API_KEY', ''),
  authDomain: getEnv('VITE_FIREBASE_AUTH_DOMAIN', ''),
  projectId: getEnv('VITE_FIREBASE_PROJECT_ID', ''),
  storageBucket: getEnv('VITE_FIREBASE_STORAGE_BUCKET', ''),
  messagingSenderId: getEnv('VITE_FIREBASE_MESSAGING_SENDER_ID', ''),
  appId: getEnv('VITE_FIREBASE_APP_ID', '')
};

// Canvas 미리보기 호환성 유지
const canvasFirebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
let db = null;
let appId = "foodie-vibe-app";

try {
  if (canvasFirebaseConfig && canvasFirebaseConfig.projectId) {
     db = getFirestore();
  } else if (firebaseConfig.projectId) {
     const app = initializeApp(firebaseConfig);
     db = getFirestore(app);
  }
} catch (e) {
  console.log("Firebase가 아직 연결되지 않았습니다 (로컬/설정전)");
}

const SURVEY_QUESTIONS = [
  { id: 'category', question: '어떤 종류의 음식이 끌리나요?', options: ['한식', '중식', '일식', '양식'] },
  { id: 'type', question: '어떤 형태가 좋으신가요?', options: ['국물류', '밥류', '면류', '기타'] },
  { id: 'taste', question: '어떤 맛을 원하시나요?', options: ['자극', '담백'] },
  { id: 'companion', question: '누구와 함께 식사하시나요?', options: ['데이트', '혼밥', '친구', '단체'] }
];

const FoodCard = ({ menu, index }) => {
  const [loadingImg, setLoadingImg] = useState(true);
  const keyword = menu.keyword ? menu.keyword.split(' ').join(',') : 'food';
  const imgSrc = `https://loremflickr.com/400/300/${keyword},food,delicious?random=${index}`;

  return (
    <div className="bg-white rounded-3xl overflow-hidden shadow-sm border border-gray-100 flex flex-col group">
      <div className="h-48 bg-gray-50 relative overflow-hidden flex items-center justify-center">
        {loadingImg && (
          <div className="absolute inset-0 flex flex-col items-center justify-center space-y-2 bg-gray-100">
            <Loader2 className="w-8 h-8 animate-spin text-rose-300" />
            <span className="text-xs font-bold text-gray-400 animate-pulse">사진을 불러오는 중...</span>
          </div>
        )}
        <img 
          src={imgSrc} 
          alt={menu.name}
          onLoad={() => setLoadingImg(false)}
          onError={(e) => { e.target.src = `https://loremflickr.com/400/300/delicious,meal?random=${index}`; }}
          className={`w-full h-full object-cover group-hover:scale-105 transition-transform duration-500 ${loadingImg ? 'opacity-0' : 'opacity-100'}`}
        />
        <div className="absolute top-3 left-3 bg-white/90 backdrop-blur px-3 py-1 rounded-full text-xs font-black text-gray-800 shadow-sm">
          추천 {index + 1}
        </div>
      </div>
      <div className="p-5">
        <h3 className="text-xl font-black text-gray-900 mb-2">{menu.name}</h3>
        <p className="text-gray-500 text-sm font-medium mb-4 leading-relaxed">{menu.description}</p>
        
        <a 
          href={`https://map.naver.com/p/search/${encodeURIComponent('주변 ' + menu.name + ' 맛집')}`}
          target="_blank" 
          rel="noreferrer"
          className="w-full flex items-center justify-center space-x-2 bg-rose-50 hover:bg-rose-100 text-rose-600 p-3 rounded-xl font-bold text-sm transition-colors group/btn"
        >
          <MapPin className="w-4 h-4 group-hover/btn:animate-bounce" />
          <span>내 주변 {menu.name} 식당 찾기</span>
          <ExternalLink className="w-3 h-3 opacity-50" />
        </a>
      </div>
    </div>
  );
};

export default function App() {
  const [user, setUser] = useState({ uid: 'guest123', name: '게스트' });
  const [currentView, setCurrentView] = useState('home');
  const [foodLogs, setFoodLogs] = useState([]);
  const [surveyAnswers, setSurveyAnswers] = useState({});
  const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);
  const [recommendations, setRecommendations] = useState([]);
  const [newFoodName, setNewFoodName] = useState('');
  const [logDate, setLogDate] = useState(new Date().toISOString().split('T')[0]);
  const [mealType, setMealType] = useState('점심');

  useEffect(() => {
    if (navigator.geolocation) {
       navigator.geolocation.getCurrentPosition(() => {}, () => {}); 
    }
  }, []);

  const handleSaveFoodLog = async () => {
    if (!newFoodName.trim()) return;
    const newLog = {
      id: Date.now().toString(),
      name: newFoodName,
      dateString: logDate,
      mealType: mealType
    };
    
    // 날짜 최신순으로 정렬해서 저장하기
    const updatedLogs = [newLog, ...foodLogs].sort((a, b) => new Date(b.dateString) - new Date(a.dateString));
    setFoodLogs(updatedLogs);
    setNewFoodName('');
    setCurrentView('home');

    if (db) {
      try {
        const logsRef = collection(db, 'users', user.uid, 'foodLogs');
        await addDoc(logsRef, { ...newLog, createdAt: serverTimestamp() });
      } catch (e) {
         console.error("Firebase 저장 실패:", e);
      }
    }
  };

  const generateRecommendations = async (answers) => {
    setCurrentView('loading');
    
    try {
      // 배포용: Vercel 환경 변수에서 구글 Gemini API 키를 가져옵니다.
      const apiKey = typeof __firebase_config !== 'undefined' ? "" : getEnv('VITE_GEMINI_API_KEY', ''); 
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${apiKey}`;
      
      // 사용자 식사 기록을 하나의 문자열로 요약 (예: "마라탕, 햄버거, 김치찌개")
      const pastMeals = foodLogs.map(log => log.name).join(', ');
      const pastMealsContext = pastMeals 
        ? `\n[사용자의 최근 식사 기록: ${pastMeals}]\n위 식사 기록을 바탕으로 평소 식성을 분석하고, 최근에 먹은 메뉴와 너무 똑같은 것은 피하되 오늘의 취향에 완벽하게 맞는 메뉴를 제안해줘.` 
        : `\n[사용자의 최근 식사 기록: 아직 없음]`;

      const prompt = `너는 메뉴 추천의 달인인 AI 푸드 소믈리에야.
사용자의 현재 취향: [카테고리: ${answers.category}], [형태: ${answers.type}], [맛: ${answers.taste}], [상황: ${answers.companion}].${pastMealsContext}

이 조건에 잘 부합하면서 군침이 도는 구체적인 음식 메뉴 5가지를 추천해줘. 

[중요 규칙]
1. 절대 세상에 없는 퓨전 요리나 새로운 메뉴를 '창작'하지 마.
2. 실제로 한국의 식당이나 프랜차이즈에서 돈을 주고 사 먹을 수 있는 대중적이고 실존하는 메뉴여야 해.
3. 그렇다고 너무 단순하게 '파스타', '김치찌개'라고만 하지 말고, 실제 메뉴판에 적혀있는 것처럼 구체적으로 추천해줘. (예: '명란 크림 파스타', '흑돼지 묵은지 김치찌개', '마라 우육면' 등)
4. 사용자가 네이버 지도에서 주변 맛집을 검색했을 때 식당이 제대로 검색될 수 있는 상용 메뉴여야 해.

반드시 아래에 정의된 JSON 배열 형태로만 응답할 것. 마크다운 기호 없이 순수 JSON만 반환해.`;

      const payload = {
        contents: [{ parts: [{ text: prompt }] }],
        generationConfig: {
          responseMimeType: "application/json",
          responseSchema: {
            type: "ARRAY",
            items: {
              type: "OBJECT",
              properties: {
                name: { type: "STRING", description: "구체적이고 맛있는 음식 이름" },
                description: { type: "STRING", description: "이 메뉴를 추천하는 감성적인 이유 (1~2줄)" },
                keyword: { type: "STRING", description: "영문 음식 이름 (이미지 생성 쿼리용, 예: spicy pork belly)" }
              },
              required: ["name", "description", "keyword"]
            }
          }
        }
      };

      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      
      const result = await response.json();
      
      if (!response.ok) {
        throw new Error(result.error?.message || "API 요청 실패");
      }

      const jsonText = result.candidates[0].content.parts[0].text;
      const aiMenus = JSON.parse(jsonText);
      
      setRecommendations(aiMenus);
      setCurrentView('result');

    } catch (error) {
      console.error("AI Recommendation Error:", error);
      setRecommendations([
        { name: "비상용 추천 맛집", description: "API 키 설정이 필요하거나 서버 오류입니다. 로컬 맛집을 검색해보세요!", keyword: "food" }
      ]);
      setCurrentView('result');
    }
  };

  const Header = ({ title, onBack }) => (
    <div className="pt-12 pb-4 px-6 flex items-center bg-white sticky top-0 z-10 border-b border-gray-50">
      {onBack && (
        <button onClick={onBack} className="p-2 -ml-2 mr-2 bg-gray-50 hover:bg-gray-100 rounded-full text-gray-600 active:scale-95 transition">
          <ChevronLeft className="w-6 h-6" />
        </button>
      )}
      <h1 className="text-xl font-bold text-gray-900">{title}</h1>
    </div>
  );

  const renderHome = () => (
    <div className="p-6 pb-24 flex flex-col space-y-8 animate-in fade-in slide-in-from-bottom-4 duration-500">
      <div className="space-y-4">
        <h2 className="text-3xl font-extrabold text-gray-900 tracking-tight leading-tight mt-6">
          아 배고파,<br/>
          <span className="text-rose-500">오늘 뭐먹지?</span>
        </h2>
      </div>

      <button 
        onClick={() => {
          setSurveyAnswers({});
          setCurrentQuestionIndex(0);
          setCurrentView('survey');
        }}
        className="w-full bg-gray-900 hover:bg-gray-800 text-white p-5 rounded-2xl font-bold text-lg flex items-center justify-between shadow-lg shadow-gray-200 active:scale-[0.98] transition-all"
      >
        <span>오늘의 메뉴 추천받기</span>
        <ArrowRight className="w-6 h-6" />
      </button>

      <div className="space-y-4 pt-4 border-t border-gray-100">
        <div className="flex items-center justify-between">
          <h3 className="font-bold text-gray-800 text-lg">최근 먹은 음식</h3>
          <button 
            onClick={() => setCurrentView('logFood')}
            className="text-sm font-semibold text-rose-500 bg-rose-50 hover:bg-rose-100 px-3 py-1.5 rounded-full flex items-center transition-colors"
          >
            <Plus className="w-4 h-4 mr-1" /> 기록하기
          </button>
        </div>
        
        {foodLogs.length === 0 ? (
          <div className="bg-gray-50 rounded-2xl p-6 text-center text-gray-500 text-sm">
            아직 기록된 식사가 없어요.<br/>메뉴를 기록하면 더 정확히 추천해드려요!
          </div>
        ) : (
          <div className="space-y-6 pb-4">
            {Object.entries(foodLogs.reduce((acc, log) => {
              // 날짜별로 데이터 예쁘게 묶어주기 (그룹핑)
              if (!acc[log.dateString]) acc[log.dateString] = [];
              acc[log.dateString].push(log);
              return acc;
            }, {})).map(([date, logs]) => (
              <div key={date} className="animate-in fade-in slide-in-from-bottom-2">
                <h4 className="text-sm font-bold text-gray-500 mb-3 flex items-center">
                  <div className="w-2 h-2 rounded-full bg-rose-300 mr-2"></div>
                  {date}
                </h4>
                <div className="space-y-3">
                  {logs.map((log, idx) => (
                    <div key={idx} className="flex items-center justify-between bg-white border border-gray-100 p-4 rounded-2xl shadow-sm">
                      <div className="flex items-center space-x-3">
                        <span className={`text-xs font-bold px-2.5 py-1 rounded-lg ${
                          log.mealType === '아침' ? 'bg-blue-50 text-blue-600' : 
                          log.mealType === '점심' ? 'bg-orange-50 text-orange-600' : 'bg-indigo-50 text-indigo-600'
                        }`}>
                          {log.mealType}
                        </span>
                        <p className="font-bold text-gray-900">{log.name}</p>
                      </div>
                      <div className="text-xl">🍽️</div>
                    </div>
                  ))}
                </div>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  );

  const renderSurvey = () => {
    const question = SURVEY_QUESTIONS[currentQuestionIndex];
    const progress = ((currentQuestionIndex + 1) / SURVEY_QUESTIONS.length) * 100;

    return (
      <div className="h-full flex flex-col bg-white">
        <Header title="취향 분석 중..." onBack={() => {
          if (currentQuestionIndex > 0) setCurrentQuestionIndex(prev => prev - 1);
          else setCurrentView('home');
        }} />
        
        <div className="w-full bg-gray-100 h-1.5">
          <div className="bg-rose-500 h-1.5 transition-all duration-300" style={{ width: `${progress}%` }} />
        </div>

        <div className="p-6 flex-1 flex flex-col justify-center mb-20 animate-in slide-in-from-right-4">
          <h2 className="text-2xl font-bold text-gray-900 mb-8 leading-tight">
            {question.question}
          </h2>
          <div className="space-y-3">
            {question.options.map((option) => (
              <button
                key={option}
                onClick={() => {
                  const newAnswers = { ...surveyAnswers, [question.id]: option };
                  setSurveyAnswers(newAnswers);
                  if (currentQuestionIndex < SURVEY_QUESTIONS.length - 1) {
                    setCurrentQuestionIndex(prev => prev + 1);
                  } else {
                    generateRecommendations(newAnswers);
                  }
                }}
                className="w-full p-5 text-left rounded-2xl border-2 border-gray-100 bg-white font-bold text-gray-700 hover:border-rose-500 hover:text-rose-600 hover:bg-rose-50 active:scale-95 transition-all shadow-sm"
              >
                {option}
              </button>
            ))}
          </div>
        </div>
      </div>
    );
  };

  const renderLoading = () => (
    <div className="h-full flex flex-col items-center justify-center p-6 space-y-6 bg-white">
      <div className="relative">
        <div className="w-24 h-24 bg-rose-50 rounded-full flex items-center justify-center animate-pulse">
          <Utensils className="w-10 h-10 text-rose-500 animate-bounce" />
        </div>
      </div>
      <div className="text-center space-y-2">
        <h2 className="text-xl font-bold text-gray-900">AI가 메뉴를 고안하는 중...</h2>
        <p className="text-gray-500 text-sm">
          뻔한 메뉴는 빼고, 특별한 식사를 준비하고 있어요!
        </p>
      </div>
    </div>
  );

  const renderResult = () => (
    <div className="h-full flex flex-col animate-in fade-in bg-gray-50 pb-24">
      <Header title="오늘의 추천 메뉴" onBack={() => setCurrentView('home')} />
      
      <div className="p-6 space-y-6 flex-1 overflow-y-auto hide-scrollbar">
        <div className="bg-white rounded-2xl p-5 shadow-sm border border-gray-100">
          <p className="text-gray-600 font-medium text-sm leading-relaxed">
            <span className="font-bold text-rose-500">{surveyAnswers.category}, {surveyAnswers.type}, {surveyAnswers.taste}, {surveyAnswers.companion}</span> 취향에 맞춰 AI가 창작한 특별한 메뉴입니다.
          </p>
        </div>

        <div className="space-y-5">
          {recommendations.map((menu, idx) => (
            <FoodCard key={idx} menu={menu} index={idx} />
          ))}
        </div>
        
        <button 
          onClick={() => {
            setSurveyAnswers({});
            setCurrentQuestionIndex(0);
            setCurrentView('survey');
          }}
          className="w-full flex items-center justify-center space-x-2 text-gray-500 font-bold py-4 active:scale-95 transition"
        >
          <RefreshCw className="w-4 h-4" />
          <span>다시 추천받기</span>
        </button>
      </div>
    </div>
  );

  const renderLogFood = () => (
    <div className="h-full flex flex-col bg-white">
      <Header title="식사 기록하기" onBack={() => setCurrentView('home')} />
      <div className="p-6 flex-1 space-y-8 overflow-y-auto hide-scrollbar">
        
        <div className="space-y-3">
          <label className="block text-sm font-bold text-gray-700">언제 드셨나요?</label>
          <input 
            type="date" 
            value={logDate}
            onChange={(e) => setLogDate(e.target.value)}
            className="w-full text-lg p-4 border-2 border-gray-100 rounded-2xl focus:border-rose-500 focus:outline-none transition-colors bg-white cursor-pointer"
          />
        </div>

        <div className="space-y-3">
          <label className="block text-sm font-bold text-gray-700">어떤 끼니였나요?</label>
          <div className="flex space-x-3">
            {['아침', '점심', '저녁'].map(type => (
              <button
                key={type}
                onClick={() => setMealType(type)}
                className={`flex-1 p-4 rounded-2xl font-bold transition-all ${
                  mealType === type 
                    ? 'bg-rose-500 text-white shadow-md shadow-rose-200' 
                    : 'bg-gray-50 text-gray-500 hover:bg-gray-100'
                }`}
              >
                {type}
              </button>
            ))}
          </div>
        </div>

        <div className="space-y-3">
          <label className="block text-sm font-bold text-gray-700">어떤 음식을 드셨나요?</label>
          <input 
            type="text" 
            value={newFoodName}
            onChange={(e) => setNewFoodName(e.target.value)}
            placeholder="예: 마라탕, 햄버거"
            className="w-full text-lg p-4 border-2 border-gray-100 rounded-2xl focus:border-rose-500 focus:outline-none transition-colors"
          />
        </div>

        <button 
          onClick={handleSaveFoodLog}
          disabled={!newFoodName.trim()}
          className="w-full bg-gray-900 disabled:bg-gray-300 text-white font-bold p-5 rounded-2xl active:scale-[0.98] transition-all shadow-lg mt-4"
        >
          기록 저장하기
        </button>
      </div>
    </div>
  );

  return (
    <div className="max-w-md mx-auto h-screen bg-white relative overflow-hidden flex flex-col font-sans border-x border-gray-100 shadow-2xl">
      <div className="flex-1 overflow-y-auto hide-scrollbar">
        {currentView === 'home' && renderHome()}
        {currentView === 'survey' && renderSurvey()}
        {currentView === 'loading' && renderLoading()}
        {currentView === 'result' && renderResult()}
        {currentView === 'logFood' && renderLogFood()}
      </div>

      {currentView === 'home' && (
        <div className="absolute bottom-0 left-0 right-0 p-4 bg-gradient-to-t from-white via-white to-transparent pointer-events-none">
          <div className="bg-gray-900 rounded-full px-6 py-4 flex justify-between items-center shadow-xl shadow-gray-200/50 pointer-events-auto">
            <button className="flex flex-col items-center text-white">
              <Home className="w-6 h-6" />
              <span className="text-[10px] font-bold mt-1">홈</span>
            </button>
            <button 
              onClick={() => {
                setSurveyAnswers({});
                setCurrentQuestionIndex(0);
                setCurrentView('survey');
              }}
              className="flex flex-col items-center text-gray-400 hover:text-white transition"
            >
              <Star className="w-6 h-6" />
              <span className="text-[10px] font-bold mt-1">AI 추천</span>
            </button>
            <button 
              onClick={() => setCurrentView('logFood')}
              className="flex flex-col items-center text-gray-400 hover:text-white transition"
            >
              <Utensils className="w-6 h-6" />
              <span className="text-[10px] font-bold mt-1">기록</span>
            </button>
          </div>
        </div>
      )}

      <style>{`
        .hide-scrollbar::-webkit-scrollbar { display: none; }
        .hide-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
      `}</style>
    </div>
  );
}