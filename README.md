import React, { useState, useEffect, useRef, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, collection, doc, setDoc, onSnapshot, query, updateDoc,
  getDoc, increment, serverTimestamp, addDoc
} from 'firebase/firestore';
import { 
  getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken
} from 'firebase/auth';
import { 
  TrendingUp, TrendingDown, Trophy, Wallet, Briefcase, Zap, 
  LineChart, X, Bitcoin, Building2, Banknote, BarChart3, Crown, 
  Map as MapIcon, Sword, Users, ShieldCheck, MessageSquare, Send,
  Pickaxe, Anchor, Dna, Clover, Gem
} from 'lucide-react';

// --- Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'rich-arena-v6-chaos';

// --- Initial Data ---
const INITIAL_STOCKS = [
  { id: 'stock_1', name: '떡상전자', price: 50000, change: 0, sector: 'IT', history: [], shares: 1000000, volatility: 0.15 },
  { id: 'stock_2', name: '화성갈끄니까', price: 120000, change: 0, sector: 'Space', history: [], shares: 500000, volatility: 0.25 },
  { id: 'stock_3', name: '탕후루소프트', price: 15000, change: 0, sector: 'Food', history: [], shares: 2000000, volatility: 0.10 },
  { id: 'coin_1', name: '비트코인', price: 80000000, change: 0, sector: 'CRYPTO', history: [], shares: 21000000, isCrypto: true, volatility: 0.50 }, 
  { id: 'coin_2', name: '도지코인', price: 200, change: 0, sector: 'MEME', history: [], shares: 10000000000, isCrypto: true, volatility: 0.60 }, 
];

const INITIAL_LANDS = Array.from({ length: 64 }, (_, i) => ({
  id: `land_${i}`,
  name: `${String.fromCharCode(65 + Math.floor(i / 8))}${i % 8 + 1}`,
  price: 20000000 + Math.floor(Math.random() * 50000000),
  owner: null,
  ownerName: null,
  color: null,
  level: 1
}));

const ITEMS = {
  stone: { name: '짱돌', price: 1000, type: 'mineral' },
  iron: { name: '철광석', price: 50000, type: 'mineral' },
  gold: { name: '금괴', price: 1000000, type: 'mineral' },
  diamond: { name: '다이아몬드', price: 50000000, type: 'mineral' },
  fish: { name: '멸치', price: 500, type: 'fish' },
  tuna: { name: '참다랑어', price: 300000, type: 'fish' },
  whale: { name: '향유고래', price: 10000000, type: 'fish' },
  chest: { name: '해적의 보물', price: 100000000, type: 'special' }
};

export default function App() {
  // --- Global State ---
  const [user, setUser] = useState(null);
  const [userName, setUserName] = useState("개미 투자자");
  
  // Assets
  const [money, setMoney] = useState(5000000); // 시작 자금 500만원
  const [loan, setLoan] = useState(0);
  const [myStocks, setMyStocks] = useState({});
  const [inventory, setInventory] = useState({}); // { itemId: count }
  
  // Progression
  const [level, setLevel] = useState(1);
  const [exp, setExp] = useState(0);
  const maxExp = level * 1000;

  // Game World
  const [stocks, setStocks] = useState(INITIAL_STOCKS);
  const [lands, setLands] = useState(INITIAL_LANDS);
  const [nations, setNations] = useState([]);
  const [myNation, setMyNation] = useState(null);
  const [myCompany, setMyCompany] = useState(null);

  // UI State
  const [activeTab, setActiveTab] = useState('market');
  const [tradeQty, setTradeQty] = useState("");
  const [chatInput, setChatInput] = useState("");
  const [chatMessages, setChatMessages] = useState([]);
  const [leaderboard, setLeaderboard] = useState([]);
  const [showRank, setShowRank] = useState(false);
  const [selectedStock, setSelectedStock] = useState(null);
  const [lottoResult, setLottoResult] = useState(null);
  const [gatherMsg, setGatherMsg] = useState(null);

  // Computed
  const netWorth = useMemo(() => {
    const stockVal = stocks.reduce((acc, s) => acc + (myStocks[s.id] || 0) * (s.price || 0), 0);
    const landVal = lands.reduce((acc, l) => acc + (l.owner === user?.uid ? l.price : 0), 0);
    const invVal = Object.entries(inventory).reduce((acc, [id, qty]) => acc + (ITEMS[id]?.price || 0) * qty, 0);
    const coVal = myCompany ? myCompany.founderValue : 0;
    return money + stockVal + landVal + invVal + coVal - loan;
  }, [money, myStocks, stocks, lands, loan, inventory, myCompany]);

  // --- 1. Firebase Auth & Listeners ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, (u) => { if (u) setUser(u); });
  }, []);

  useEffect(() => {
    if (!user) return;
    
    // User Data Sync
    const userRef = doc(db, 'artifacts', appId, 'users', user.uid, 'account', 'data');
    getDoc(userRef).then(s => {
      if (s.exists()) {
        const d = s.data();
        if(d.money !== undefined) setMoney(d.money);
        if(d.myStocks) setMyStocks(d.myStocks);
        if(d.userName) setUserName(d.userName);
        if(d.loan) setLoan(d.loan);
        if(d.inventory) setInventory(d.inventory);
        if(d.level) setLevel(d.level);
        if(d.exp) setExp(d.exp);
        if(d.myNation) setMyNation(d.myNation);
        if(d.myCompany) setMyCompany(d.myCompany);
      }
    });

    // Market Sync
    const marketRef = doc(db, 'artifacts', appId, 'public', 'data', 'game_data', 'market_v2');
    const unsubMarket = onSnapshot(marketRef, (snap) => {
      if (snap.exists()) {
        const d = snap.data();
        if (d.stocks) setStocks(d.stocks);
        if (d.lands) setLands(d.lands);
        if (d.nations) setNations(d.nations);
      } else {
        setDoc(marketRef, { stocks: INITIAL_STOCKS, lands: INITIAL_LANDS, nations: [] });
      }
    });

    // Chat Sync
    const chatRef = collection(db, 'artifacts', appId, 'public', 'data', 'chat');
    const unsubChat = onSnapshot(query(chatRef), (s) => {
       setChatMessages(s.docs.map(d => d.data()).sort((a,b) => a.ts - b.ts).slice(-30));
    });

    // Ranking Sync
    const rankRef = collection(db, 'artifacts', appId, 'public', 'data', 'ranking');
    const unsubRank = onSnapshot(query(rankRef), (s) => {
      setLeaderboard(s.docs.map(d => d.data()).sort((a,b) => b.total - a.total).slice(0, 10));
    });

    return () => { unsubMarket(); unsubChat(); unsubRank(); };
  }, [user]);

  // --- 2. Auto Save ---
  useEffect(() => {
    if (!user) return;
    const t = setTimeout(async () => {
      const userRef = doc(db, 'artifacts', appId, 'users', user.uid, 'account', 'data');
      await setDoc(userRef, { 
        money, myStocks, userName, loan, inventory, level, exp, myNation, myCompany, total: netWorth 
      }, { merge: true });
      
      const rankDoc = doc(db, 'artifacts', appId, 'public', 'data', 'ranking', user.uid);
      await setDoc(rankDoc, { uid: user.uid, name: userName, total: netWorth, level }, { merge: true });
    }, 2000);
    return () => clearTimeout(t);
  }, [money, myStocks, loan, inventory, level, exp, netWorth]);

  // --- 3. Market Engine (High Volatility) ---
  useEffect(() => {
    if (!user) return;
    const interval = setInterval(() => {
      if (Math.random() > 0.7) updateMarket(); // Only update sometimes to prevent too many writes
    }, 1500);
    return () => clearInterval(interval);
  }, [user, stocks]);

  const updateMarket = async () => {
    const marketRef = doc(db, 'artifacts', appId, 'public', 'data', 'game_data', 'market_v2');
    const snap = await getDoc(marketRef).catch(() => null);
    if (!snap || !snap.exists()) return;

    const currentStocks = snap.data().stocks || INITIAL_STOCKS;
    
    const newStocks = currentStocks.map(s => {
      // 변동성 로직 강화
      // 기본 변동성 계수 + 랜덤 폭주
      const isCrypto = s.isCrypto;
      const baseVol = isCrypto ? 0.05 : 0.02; // 틱당 기본 5% / 2%
      const chaos = Math.random() < 0.1 ? (isCrypto ? 0.5 : 0.3) : 0; // 10% 확률로 50% / 30% 폭등락
      
      let changePercent = (Math.random() - 0.5) * 2 * (baseVol + chaos); 
      
      // 가격 적용
      let newPrice = Math.floor(s.price * (1 + changePercent));
      if (newPrice < 10) newPrice = 10; // 하한선

      // 기록 업데이트
      const history = [...(s.history || []), newPrice].slice(-30);
      
      return { 
        ...s, 
        price: newPrice, 
        change: changePercent * 100,
        history 
      };
    });

    await updateDoc(marketRef, { stocks: newStocks, lastUpdate: serverTimestamp() }).catch(()=>{});
  };

  // --- 4. Game Actions ---

  // Trading
  const trade = (type, stock) => {
    let q = parseInt(tradeQty);
    if (!q || isNaN(q)) q = type === 'buy' ? Math.floor(money / stock.price) : (myStocks[stock.id] || 0);
    if (q <= 0) return;

    if (type === 'buy') {
      if (money < stock.price * q) return alert("잔액 부족");
      setMoney(m => m - stock.price * q);
      setMyStocks(p => ({ ...p, [stock.id]: (p[stock.id] || 0) + q }));
    } else {
      if ((myStocks[stock.id] || 0) < q) return alert("주식 부족");
      setMoney(m => m + stock.price * q);
      setMyStocks(p => ({ ...p, [stock.id]: p[stock.id] - q }));
    }
    setTradeQty("");
  };

  // Jobs (Mining/Fishing)
  const handleGather = (type) => {
    // 쿨타임/에너지 대신 단순 클릭 노동
    const roll = Math.random() * (1 + level * 0.1); // 레벨 높을수록 행운 증가
    let item = null;
    let msg = "";

    if (type === 'mining') {
      if (roll > 50) item = 'diamond';
      else if (roll > 20) item = 'gold';
      else if (roll > 5) item = 'iron';
      else item = 'stone';
    } else if (type === 'fishing') {
      if (roll > 80) item = 'chest';
      else if (roll > 40) item = 'whale';
      else if (roll > 10) item = 'tuna';
      else item = 'fish';
    }

    if (item) {
      setInventory(prev => ({ ...prev, [item]: (prev[item] || 0) + 1 }));
      const gainedExp = ITEMS[item].price > 100000 ? 50 : 10;
      gainExp(gainedExp);
      setGatherMsg(`+ ${ITEMS[item].name} 획득!`);
      setTimeout(() => setGatherMsg(null), 1000);
    }
  };

  const sellItem = (itemId) => {
    const qty = inventory[itemId];
    if (!qty) return;
    setMoney(m => m + ITEMS[itemId].price * qty);
    setInventory(prev => ({ ...prev, [itemId]: 0 }));
  };

  const gainExp = (amount) => {
    let newExp = exp + amount;
    if (newExp >= maxExp) {
      setExp(newExp - maxExp);
      setLevel(l => l + 1);
      alert(`🎉 LEVEL UP! Lv.${level + 1} 달성! 채집 행운이 증가합니다.`);
    } else {
      setExp(newExp);
    }
  };

  // Lotto
  const buyLotto = () => {
    const cost = 1000000;
    if (money < cost) return alert("돈이 없습니다. (100만원 필요)");
    setMoney(m => m - cost);
    
    // 즉석 추첨 로직
    const winRate = Math.random();
    let prize = 0;
    let rank = "";

    if (winRate < 0.00001) { prize = 100000000000; rank = "1등 (1000억)"; }
    else if (winRate < 0.001) { prize = 5000000000; rank = "2등 (50억)"; }
    else if (winRate < 0.05) { prize = 100000000; rank = "3등 (1억)"; }
    else if (winRate < 0.2) { prize = 5000000; rank = "4등 (500만)"; }
    else { prize = 0; rank = "꽝"; }

    setLottoResult({ rank, prize });
    if (prize > 0) {
      setMoney(m => m + prize);
      gainExp(100);
    }
  };

  // Nation & Company
  const upgradeNation = async (type) => {
    if (!myNation) return;
    const cost = 500000000; // 5억
    if (money < cost) return alert("국고가 부족합니다.");
    
    setMoney(m => m - cost);
    const updated = { ...myNation, [type]: (myNation[type] || 0) + 10 };
    updated.power = (updated.military || 0) + (updated.economy || 0) + (updated.tech || 0);
    setMyNation(updated);
    
    const marketRef = doc(db, 'artifacts', appId, 'public', 'data', 'game_data', 'market_v2');
    // 실제로는 전체 nations 배열을 가져와서 업데이트해야 함 (약식 구현)
    const newNations = nations.map(n => n.owner === user.uid ? updated : n);
    if (!nations.find(n => n.owner === user.uid)) newNations.push(updated);
    
    await updateDoc(marketRef, { nations: newNations });
  };

  const createNation = () => {
    if (money < 10000000000) return alert("건국 자금 100억원이 필요합니다.");
    setMoney(m => m - 10000000000);
    setMyNation({ name: `${userName} 제국`, owner: user.uid, military: 10, economy: 10, tech: 10, power: 30 });
  };

  // Chat
  const sendChat = async () => {
    if (!chatInput.trim()) return;
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'chat'), {
      sender: userName, uid: user.uid, text: chatInput, ts: Date.now()
    });
    setChatInput("");
  };

  // Chart Helper
  const getChartPoints = (history) => {
    if (!history || history.length < 2) return "";
    const min = Math.min(...history);
    const max = Math.max(...history);
    const range = max - min || 1;
    return history.map((p, i) => 
      `${(i / (history.length - 1)) * 300},${100 - ((p - min) / range) * 80}`
    ).join(' ');
  };

  return (
    <div className="min-h-screen bg-[#0f172a] text-slate-100 font-sans flex flex-col overflow-hidden selection:bg-rose-500 selection:text-white">
      {/* --- Header --- */}
      <header className="bg-slate-900/80 backdrop-blur border-b border-slate-800 p-4 flex justify-between items-center z-20">
        <div className="flex items-center gap-4">
          <div className="bg-gradient-to-br from-indigo-500 to-purple-600 p-2.5 rounded-xl shadow-lg shadow-indigo-500/20">
            <Zap size={20} className="text-white" fill="currentColor"/>
          </div>
          <div>
            <div className="flex items-baseline gap-2">
              <input value={userName} onChange={e=>setUserName(e.target.value)} className="bg-transparent font-black text-lg w-32 outline-none focus:text-indigo-400 transition-colors" />
              <span className="text-xs font-bold text-amber-400 bg-amber-400/10 px-2 py-0.5 rounded">Lv.{level}</span>
            </div>
            <div className="w-32 h-1.5 bg-slate-800 rounded-full mt-1 overflow-hidden">
              <div className="h-full bg-indigo-500 transition-all duration-500" style={{width: `${(exp/maxExp)*100}%`}}></div>
            </div>
          </div>
        </div>
        
        <div className="flex items-center gap-6">
          <div className="text-right hidden md:block">
            <p className="text-[10px] text-slate-500 font-black uppercase tracking-widest">Total Net Worth</p>
            <p className="text-xl font-mono font-black text-emerald-400">₩ {Math.floor(netWorth).toLocaleString()}</p>
          </div>
          <button onClick={() => setShowRank(true)} className="bg-slate-800 p-3 rounded-xl hover:bg-amber-500 hover:text-slate-900 transition-all">
            <Trophy size={18} />
          </button>
        </div>
      </header>

      <div className="flex-1 flex overflow-hidden">
        {/* --- Sidebar --- */}
        <nav className="w-20 md:w-64 bg-slate-900/50 border-r border-slate-800 flex flex-col gap-2 p-3 overflow-y-auto">
          <div className="bg-slate-800 p-4 rounded-2xl mb-4 border border-slate-700 hidden md:block">
            <p className="text-[9px] text-slate-500 font-black uppercase mb-1">Available Cash</p>
            <p className="text-lg font-mono font-black text-indigo-300 truncate">₩ {money.toLocaleString()}</p>
          </div>

          {[
            { id: 'market', icon: BarChart3, label: '거래소' },
            { id: 'jobs', icon: Pickaxe, label: '작업장' },
            { id: 'gacha', icon: Dna, label: '뽑기/복권' },
            { id: 'land', icon: MapIcon, label: '부동산' },
            { id: 'nation', icon: ShieldCheck, label: '국가' },
            { id: 'portfolio', icon: Wallet, label: '자산' },
          ].map(menu => (
            <button key={menu.id} onClick={() => setActiveTab(menu.id)} 
              className={`flex items-center gap-4 p-4 rounded-2xl transition-all ${activeTab === menu.id ? 'bg-indigo-600 shadow-lg shadow-indigo-500/20 text-white' : 'text-slate-500 hover:bg-slate-800 hover:text-slate-200'}`}>
              <menu.icon size={20} />
              <span className="hidden md:block font-bold text-xs">{menu.label}</span>
            </button>
          ))}
        </nav>

        {/* --- Main Content --- */}
        <main className="flex-1 p-4 md:p-8 overflow-y-auto relative bg-[radial-gradient(circle_at_top,_var(--tw-gradient-stops))] from-indigo-950/30 via-slate-950 to-slate-950">
          
          {/* Market Tab */}
          {activeTab === 'market' && (
            <div className="max-w-5xl mx-auto space-y-6 animate-in fade-in slide-in-from-bottom-4 duration-500">
              <div className="flex gap-4 mb-6">
                <input type="number" value={tradeQty} onChange={e => setTradeQty(e.target.value)} placeholder="주문 수량" className="bg-slate-900 border border-slate-700 p-4 rounded-2xl outline-none focus:border-indigo-500 font-mono text-lg flex-1 transition-all" />
                <button onClick={() => setTradeQty("")} className="px-6 rounded-2xl bg-slate-800 font-bold text-xs uppercase hover:bg-slate-700">Clear</button>
              </div>

              <div className="grid grid-cols-1 gap-4">
                {stocks.map(s => (
                  <div key={s.id} onClick={() => setSelectedStock(s)} className="bg-slate-900/80 border border-slate-800 p-6 rounded-[32px] hover:border-indigo-500/50 transition-all cursor-pointer group relative overflow-hidden">
                    {/* Background Chart */}
                    <div className="absolute bottom-0 left-0 w-full h-24 opacity-10 pointer-events-none">
                      <svg width="100%" height="100%" viewBox="0 0 300 100" preserveAspectRatio="none">
                        <polyline points={getChartPoints(s.history)} fill="none" stroke={s.change >= 0 ? '#10b981' : '#f43f5e'} strokeWidth="2" vectorEffect="non-scaling-stroke" />
                      </svg>
                    </div>

                    <div className="flex justify-between items-center relative z-10">
                      <div className="flex items-center gap-4">
                        <div className={`w-12 h-12 rounded-2xl flex items-center justify-center ${s.isCrypto ? 'bg-amber-500/20 text-amber-500' : 'bg-indigo-500/20 text-indigo-500'}`}>
                          {s.isCrypto ? <Bitcoin size={24}/> : <Building2 size={24}/>}
                        </div>
                        <div>
                          <h3 className="text-xl font-black italic">{s.name}</h3>
                          <p className="text-[10px] font-black text-slate-500 uppercase tracking-widest">{s.sector} • {s.isCrypto ? '±50% RISK' : '±30% RISK'}</p>
                        </div>
                      </div>
                      <div className="text-right">
                        <p className="text-2xl font-mono font-black">₩ {s.price.toLocaleString()}</p>
                        <p className={`text-sm font-bold ${s.change >= 0 ? 'text-emerald-400' : 'text-rose-500'}`}>
                          {s.change > 0 ? '+' : ''}{s.change.toFixed(2)}%
                        </p>
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          )}

          {/* Jobs Tab (Mining & Fishing) */}
          {activeTab === 'jobs' && (
            <div className="max-w-4xl mx-auto space-y-8 animate-in zoom-in-95 duration-300">
              <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                {/* Mining */}
                <div onClick={() => handleGather('mining')} className="bg-slate-900 p-8 rounded-[40px] border-4 border-slate-800 hover:border-amber-700 cursor-pointer active:scale-95 transition-all relative overflow-hidden group">
                  <div className="absolute top-0 right-0 p-10 opacity-5 group-hover:opacity-10 transition-opacity"><Pickaxe size={200}/></div>
                  <Pickaxe size={48} className="text-amber-600 mb-6 group-hover:rotate-12 transition-transform"/>
                  <h2 className="text-3xl font-black italic mb-2">MINING</h2>
                  <p className="text-slate-500 text-xs font-bold uppercase mb-8">Click to mine minerals</p>
                  <div className="bg-slate-950 p-4 rounded-2xl border border-slate-800">
                    <p className="text-[10px] text-slate-500 font-black uppercase mb-2">Mining Loot</p>
                    <div className="grid grid-cols-2 gap-2">
                       {['stone', 'iron', 'gold', 'diamond'].map(k => (
                         inventory[k] > 0 && 
                         <div key={k} onClick={(e)=>{e.stopPropagation(); sellItem(k);}} className="bg-slate-900 p-2 rounded-lg text-xs flex justify-between hover:bg-slate-800">
                           <span>{ITEMS[k].name}</span> <span className="font-mono text-amber-500">x{inventory[k]}</span>
                         </div>
                       ))}
                    </div>
                  </div>
                </div>

                {/* Fishing */}
                <div onClick={() => handleGather('fishing')} className="bg-slate-900 p-8 rounded-[40px] border-4 border-slate-800 hover:border-blue-700 cursor-pointer active:scale-95 transition-all relative overflow-hidden group">
                  <div className="absolute top-0 right-0 p-10 opacity-5 group-hover:opacity-10 transition-opacity"><Anchor size={200}/></div>
                  <Anchor size={48} className="text-blue-500 mb-6 group-hover:-rotate-12 transition-transform"/>
                  <h2 className="text-3xl font-black italic mb-2">FISHING</h2>
                  <p className="text-slate-500 text-xs font-bold uppercase mb-8">Click to catch fish</p>
                  <div className="bg-slate-950 p-4 rounded-2xl border border-slate-800">
                    <p className="text-[10px] text-slate-500 font-black uppercase mb-2">Fishing Loot</p>
                    <div className="grid grid-cols-2 gap-2">
                       {['fish', 'tuna', 'whale', 'chest'].map(k => (
                         inventory[k] > 0 && 
                         <div key={k} onClick={(e)=>{e.stopPropagation(); sellItem(k);}} className="bg-slate-900 p-2 rounded-lg text-xs flex justify-between hover:bg-slate-800">
                           <span>{ITEMS[k].name}</span> <span className="font-mono text-blue-400">x{inventory[k]}</span>
                         </div>
                       ))}
                    </div>
                  </div>
                </div>
              </div>
              
              {gatherMsg && (
                <div className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-black/80 text-white px-6 py-3 rounded-full font-black animate-bounce pointer-events-none z-50">
                  {gatherMsg}
                </div>
              )}
            </div>
          )}

          {/* Gacha & Lotto Tab */}
          {activeTab === 'gacha' && (
            <div className="max-w-4xl mx-auto space-y-8 animate-in fade-in duration-500">
              <div className="bg-gradient-to-r from-violet-900 to-fuchsia-900 p-10 rounded-[48px] shadow-2xl text-center relative overflow-hidden">
                <Clover size={100} className="mx-auto mb-6 text-white/20 absolute -top-4 -right-4 rotate-12"/>
                <h2 className="text-4xl font-black italic text-white mb-2 uppercase">Arena Lottery</h2>
                <p className="text-fuchsia-200 text-sm font-bold mb-8">1등 당첨금: 1,000억 원 (즉석 추첨)</p>
                
                <div className="bg-black/30 backdrop-blur p-8 rounded-[32px] inline-block mb-8 border border-white/10">
                  {lottoResult ? (
                    <div className="animate-in zoom-in">
                       <p className="text-sm font-black text-fuchsia-300 mb-2">RESULT</p>
                       <p className="text-3xl font-black text-white">{lottoResult.rank}</p>
                       {lottoResult.prize > 0 && <p className="text-emerald-400 font-mono font-bold mt-2">+ ₩{lottoResult.prize.toLocaleString()}</p>}
                    </div>
                  ) : (
                    <p className="text-2xl font-black text-white/50">???</p>
                  )}
                </div>
                
                <br/>
                <button onClick={buyLotto} className="bg-white text-fuchsia-900 px-10 py-4 rounded-2xl font-black text-lg hover:scale-105 active:scale-95 transition-all shadow-xl">
                  구매하기 (₩1,000,000)
                </button>
              </div>

               <div className="bg-slate-900 p-10 rounded-[48px] border border-slate-800 text-center relative overflow-hidden">
                 <h3 className="text-2xl font-black italic mb-4 flex items-center justify-center gap-2"><Gem className="text-sky-400"/> Random Gacha Box</h3>
                 <p className="text-slate-500 text-xs mb-6">랜덤하게 거액의 자산을 획득할 수 있습니다.</p>
                 <button onClick={() => { if(money>=10000000) { setMoney(m=>m-10000000); handleGather('fishing'); } else alert('돈 부족'); }} className="bg-slate-800 text-sky-400 border border-sky-500/30 px-8 py-3 rounded-xl font-bold hover:bg-sky-900/20 transition-all">
                    1,000만 Gacha
                 </button>
               </div>
            </div>
          )}
          
          {/* Nation Tab */}
          {activeTab === 'nation' && (
             <div className="max-w-4xl mx-auto space-y-6">
                {!myNation ? (
                  <div className="text-center p-20 bg-slate-900 rounded-[40px] border border-slate-800">
                    <ShieldCheck size={64} className="mx-auto text-slate-600 mb-6"/>
                    <h2 className="text-3xl font-black mb-4">국가 건국</h2>
                    <p className="text-slate-400 mb-8">100억 원으로 당신만의 국가를 세우고 통치하세요.</p>
                    <button onClick={createNation} className="bg-indigo-600 px-8 py-4 rounded-2xl font-black text-white hover:bg-indigo-500">건국하기 (₩100억)</button>
                  </div>
                ) : (
                  <div className="bg-slate-900 p-10 rounded-[40px] border border-slate-800">
                    <div className="flex justify-between items-start mb-10">
                       <div>
                         <h2 className="text-4xl font-black italic text-amber-500 mb-2">{myNation.name}</h2>
                         <p className="text-xs font-bold text-slate-500">POWER: {myNation.power}</p>
                       </div>
                       <div className="bg-slate-950 p-4 rounded-2xl">
                          <Crown className="text-amber-500" size={32} />
                       </div>
                    </div>
                    <div className="grid grid-cols-3 gap-4">
                       {['military', 'economy', 'tech'].map(t => (
                         <button key={t} onClick={()=>upgradeNation(t)} className="bg-slate-800 p-6 rounded-3xl hover:bg-indigo-600/20 hover:border-indigo-500 border border-transparent transition-all group">
                            <p className="text-[10px] text-slate-500 uppercase font-black mb-2 group-hover:text-indigo-400">{t}</p>
                            <p className="text-2xl font-mono font-black">{myNation[t] || 0}</p>
                            <p className="text-[9px] text-slate-600 mt-2">비용: 5억</p>
                         </button>
                       ))}
                    </div>
                  </div>
                )}
             </div>
          )}

          {/* Portfolio & Bank */}
          {activeTab === 'portfolio' && (
             <div className="max-w-4xl mx-auto space-y-6">
                <div className="bg-slate-900 p-8 rounded-[40px] flex justify-between items-center border border-slate-800">
                   <div>
                     <p className="text-xs font-bold text-slate-500 mb-1">총 순자산</p>
                     <p className="text-4xl font-black font-mono text-emerald-400">₩ {Math.floor(netWorth).toLocaleString()}</p>
                   </div>
                   <div className="text-right">
                     <p className="text-xs font-bold text-slate-500 mb-1">부채</p>
                     <p className="text-2xl font-black font-mono text-rose-500">₩ {loan.toLocaleString()}</p>
                   </div>
                </div>
                <div className="grid grid-cols-2 gap-4">
                   <button onClick={()=>{setLoan(l=>l+100000000); setMoney(m=>m+100000000)}} className="bg-slate-800 p-6 rounded-3xl font-bold hover:bg-slate-700">대출받기 (+1억)</button>
                   <button onClick={()=>{if(money>=loan){setMoney(m=>m-loan); setLoan(0)}} } className="bg-slate-800 p-6 rounded-3xl font-bold text-rose-400 hover:bg-rose-900/20">전액상환</button>
                </div>
             </div>
          )}
        </main>
      </div>

      {/* --- Floating Elements --- */}
      
      {/* Chat Widget */}
      <div className="fixed bottom-6 right-6 w-80 bg-slate-900/90 backdrop-blur border border-slate-700 rounded-3xl overflow-hidden shadow-2xl flex flex-col h-64 z-40">
        <div className="bg-slate-800 p-3 text-[10px] font-black uppercase text-slate-400 flex items-center gap-2">
          <MessageSquare size={12}/> Live Chat
        </div>
        <div className="flex-1 overflow-y-auto p-3 space-y-2 flex flex-col-reverse custom-scrollbar">
          {[...chatMessages].reverse().map((msg, i) => (
            <div key={i} className={`text-xs ${msg.uid === user?.uid ? 'text-right' : 'text-left'}`}>
              <span className="font-bold text-[9px] text-slate-500 block mb-0.5">{msg.sender}</span>
              <span className={`inline-block px-3 py-1.5 rounded-xl ${msg.uid === user?.uid ? 'bg-indigo-600 text-white' : 'bg-slate-800 text-slate-300'}`}>
                {msg.text}
              </span>
            </div>
          ))}
        </div>
        <div className="p-2 bg-slate-800 flex gap-2">
          <input value={chatInput} onChange={e=>setChatInput(e.target.value)} onKeyPress={e=>e.key==='Enter'&&sendChat()} className="flex-1 bg-slate-900 rounded-xl px-3 text-xs outline-none border border-slate-700" placeholder="메시지 전송..." />
          <button onClick={sendChat} className="bg-indigo-600 p-2 rounded-xl text-white"><Send size={14}/></button>
        </div>
      </div>

      {/* Stock Modal */}
      {selectedStock && (
        <div className="fixed inset-0 bg-black/80 backdrop-blur-sm flex items-center justify-center p-4 z-50">
          <div className="bg-slate-900 w-full max-w-2xl rounded-[40px] border border-slate-800 shadow-2xl p-8 animate-in zoom-in duration-200">
             <div className="flex justify-between items-center mb-8">
               <h2 className="text-3xl font-black italic">{selectedStock.name}</h2>
               <button onClick={()=>setSelectedStock(null)} className="p-2 bg-slate-800 rounded-full hover:bg-slate-700"><X size={20}/></button>
             </div>
             <div className="flex gap-4 mb-8">
                <button onClick={()=>{handleTrade('buy', selectedStock); setSelectedStock(null)}} className="flex-1 bg-indigo-600 py-4 rounded-2xl font-black text-lg hover:bg-indigo-500">매수 (Buy)</button>
                <button onClick={()=>{handleTrade('sell', selectedStock); setSelectedStock(null)}} className="flex-1 bg-rose-600 py-4 rounded-2xl font-black text-lg hover:bg-rose-500">매도 (Sell)</button>
             </div>
             <p className="text-center text-xs text-slate-500">주문 수량을 비워두면 최대 가능 수량으로 거래됩니다.</p>
          </div>
        </div>
      )}

      {/* Ranking Modal */}
      {showRank && (
        <div className="fixed inset-0 bg-black/80 backdrop-blur-sm flex items-center justify-center p-4 z-50">
          <div className="bg-slate-900 w-full max-w-md rounded-[40px] border border-slate-800 p-6 animate-in fade-in zoom-in">
             <div className="flex justify-between mb-6">
                <h2 className="text-xl font-black flex items-center gap-2"><Trophy className="text-amber-500"/> Leaderboard</h2>
                <button onClick={()=>setShowRank(false)}><X/></button>
             </div>
             <div className="space-y-2">
               {leaderboard.map((p, i) => (
                 <div key={i} className="flex justify-between items-center bg-slate-800 p-3 rounded-xl border border-slate-700">
                    <span className="font-bold w-6 text-center text-slate-500">{i+1}</span>
                    <span className="flex-1 font-bold">{p.name} <span className="text-[9px] text-amber-500 bg-amber-900/20 px-1 rounded ml-1">Lv.{p.level||1}</span></span>
                    <span className="font-mono text-emerald-400">₩{(p.total/100000000).toFixed(1)}억</span>
                 </div>
               ))}
             </div>
          </div>
        </div>
      )}
    </div>
  );
}
