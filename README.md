<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>스마트 스터디 플래너</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
   <style>
        @import url('https://fonts.googleapis.com/css2?family=Gowun+Dodum:wght@400;700&family=Orbitron:wght@400;700&display=swap');
        
        body { 
            font-family: 'Gowun Dodum', sans-serif; 
            background-color: #F0F9FF; 
            margin: 0; 
            overflow: hidden; 
        }
        .digital-font { font-family: 'Orbitron', sans-serif; }
        .checklist-font { font-family: 'Gulim', 'Malgun Gothic', sans-serif; font-weight: 900; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .line-clamp-2 { 
            display: -webkit-box; 
            -webkit-line-clamp: 2; 
            -webkit-box-orient: vertical; 
            overflow: hidden; 
        }
        
        /* Binder Ring Decoration */
        .binder-ring {
            width: 40px;
            height: 12px;
            background: linear-gradient(to right, #dbeafe, #ffffff66);
            border-radius: 9999px;
            margin-left: -20px;
            box-shadow: 0 1px 2px rgba(0,0,0,0.05);
            border: 1px solid rgba(255,255,255,0.2);
        }
    </style>
</head>
<body>
    <div id="root"></div>

   <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        // --- Firebase Scripts (Using Modules via script tags is tricky, using the compat versions for HTML) ---
        // Note: For this HTML version, we assume Firebase is loaded via CDN
    </script>

    <!-- Firebase SDKs -->
   <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>

   <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        // Firebase Configuration
        const firebaseConfig = {
            apiKey: "AIzaSyBv14g9crV8vGbobK5cdVxwqlrr0EiM0fA",
            authDomain: "bokk-55eae.firebaseapp.com",
            projectId: "bokk-55eae",
            storageBucket: "bokk-55eae.firebasestorage.app",
            messagingSenderId: "757990079253",
            appId: "1:757990079253:web:f3dbf171b2137f436bc71c",
            measurementId: "G-2PMDNT4LM2"
        };

        // Initialize Firebase
        firebase.initializeApp(firebaseConfig);
        const auth = firebase.auth();
        const db = firebase.firestore();
        const APP_ID = 'planner-app-v1';

        const App = () => {
            const [user, setUser] = useState(null);
            const [view, setView] = useState('daily');
            const [records, setRecords] = useState({});
            const [loading, setLoading] = useState(true);
            const [dDayTarget, setDDayTarget] = useState('');
            
            const getTodayStr = () => {
                const d = new Date();
                return `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
            };

            const [currentDate, setCurrentDate] = useState(getTodayStr());
            const [isTimerRunning, setIsTimerRunning] = useState(false);
            const [seconds, setSeconds] = useState(0);
            const timerRef = useRef(null);

            const currentRecord = records[currentDate] || { 
                date: currentDate, resolution: "", todos: [], totalTime: 0 
            };

            useEffect(() => {
                const unsubscribeAuth = auth.onAuthStateChanged(async (currentUser) => {
                    if (!currentUser) {
                        await auth.signInAnonymously();
                    } else {
                        setUser(currentUser);
                        
                        // Sync D-Day
                        const dDayDocRef = db.collection('artifacts').doc(APP_ID).collection('users').doc(currentUser.uid).collection('settings').doc('dday');
                        dDayDocRef.onSnapshot(docSnap => {
                            if (docSnap.exists) setDDayTarget(docSnap.data().target || '');
                        });

                        // Sync Records
                        const recordsColRef = db.collection('artifacts').doc(APP_ID).collection('users').doc(currentUser.uid).collection('records');
                        recordsColRef.onSnapshot(querySnapshot => {
                            const data = {};
                            querySnapshot.forEach(doc => { data[doc.id] = doc.data(); });
                            setRecords(data);
                            setLoading(false);
                        });
                    }
                });
                return () => unsubscribeAuth();
            }, []);

            useEffect(() => {
                setSeconds(currentRecord.totalTime || 0);
                setIsTimerRunning(false);
                if (timerRef.current) clearInterval(timerRef.current);
            }, [currentDate, currentRecord.totalTime]);

            const saveRecordToCloud = async (updates) => {
                if (!user) return;
                const recordRef = db.collection('artifacts').doc(APP_ID).collection('users').doc(user.uid).collection('records').doc(currentDate);
                await recordRef.set({ ...currentRecord, ...updates }, { merge: true });
            };

            const handleDDayChange = async (val) => {
                setDDayTarget(val);
                if (!user) return;
                const dDayDocRef = db.collection('artifacts').doc(APP_ID).collection('users').doc(user.uid).collection('settings').doc('dday');
                await dDayDocRef.set({ target: val });
            };

            const formatDisplayDate = (dateStr) => {
                const year = dateStr.substring(0, 4);
                const month = dateStr.substring(4, 6);
                const day = dateStr.substring(6, 8);
                const d = new Date(`${year}-${month}-${day}`);
                const days = ["일", "월", "화", "수", "목", "금", "토"];
                return `${year}/${month}/${day}/(${days[d.getDay()]})`;
            };

            const toggleTimer = () => {
                if (isTimerRunning) {
                    clearInterval(timerRef.current);
                    saveRecordToCloud({ totalTime: seconds });
                } else {
                    timerRef.current = setInterval(() => setSeconds(prev => prev + 1), 1000);
                }
                setIsTimerRunning(!isTimerRunning);
            };

            const resetTimer = () => {
                if (timerRef.current) clearInterval(timerRef.current);
                setIsTimerRunning(false);
                setSeconds(0);
                saveRecordToCloud({ totalTime: 0 });
            };

            const calculateDDay = () => {
                if (!dDayTarget) return "D-?";
                const target = new Date(dDayTarget);
                const today = new Date();
                today.setHours(0,0,0,0);
                const diff = Math.ceil((target - today) / (1000 * 60 * 60 * 24));
                return diff === 0 ? "D-Day" : diff > 0 ? `D-${diff}` : `D+${Math.abs(diff)}`;
            };

            const changeDate = (offset) => {
                if (isTimerRunning) toggleTimer();
                const d = new Date(currentDate.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'));
                d.setDate(d.getDate() + offset);
                const newDate = `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
                setCurrentDate(newDate);
                setView('daily');
            };

            const theme = { primary: '#BAE6FD', secondary: '#E0F2FE', point: '#0284C7', sidebar: '#0C4A6E', timerBg: '#0C4A6E' };
            const timetable = [
                ["자율", "문권", "연구", "문박", "B"], ["E", "A", "D", "C", "영이"], ["A", "B", "영이", "스최", "대손"],
                ["스최", "E", "대손", "대손", "E"], ["영유", "대손", "문박", "D", "C"], ["D", "영유", "B", "A", "창체"],
                ["문권", "C", "창체", "창체", "창체"]
            ];

            if (loading) return (
                <div className="h-screen bg-blue-50 flex flex-col items-center justify-center font-bold text-blue-900 gap-4">
                    <div className="w-12 h-12 border-4 border-blue-200 border-t-blue-600 rounded-full animate-spin"></div>
                    <p>플래너 데이터를 불러오는 중...</p>
                </div>
            );

            return (
                <div className="h-screen bg-[#F0F9FF] flex items-center justify-center p-4 overflow-hidden">
                    <div className="w-full max-w-[1100px] h-full max-h-[850px] bg-white shadow-2xl flex relative border-l-[30px] overflow-hidden rounded-sm" style={{ borderColor: theme.sidebar }}>
                        
                        <div className="absolute left-0 top-0 bottom-0 w-8 flex flex-col justify-around py-6 z-10 pointer-events-none">
                            {[...Array(14)].map((_, i) => <div key={i} className="binder-ring" />)}
                        </div>

                        <div className="flex-1 flex flex-col ml-6 overflow-hidden">
                            <div className="px-6 py-2.5 flex justify-between items-center border-b border-blue-50 shrink-0 bg-white">
                                <div className="flex items-center gap-6">
                                    <div className="flex items-center gap-1.5">
                                        <button onClick={() => changeDate(-1)} className="p-1 hover:bg-blue-50 rounded-full transition-colors"><i data-lucide="chevron-left" className="w-5 h-5 text-blue-300"></i></button>
                                        <h1 className="text-xl font-black tracking-tight text-gray-900 min-w-[140px] text-center">{formatDisplayDate(currentDate)}</h1>
                                        <button onClick={() => changeDate(1)} className="p-1 hover:bg-blue-50 rounded-full transition-colors"><i data-lucide="chevron-right" className="w-5 h-5 text-blue-300"></i></button>
                                    </div>
                                    <div className="flex items-center gap-2">
                                        <div className="flex items-center gap-2 px-3 py-1 rounded-full border shadow-sm" style={{ backgroundColor: theme.secondary, borderColor: theme.primary }}>
                                            <div className="text-lg font-black leading-none tracking-tighter" style={{ color: theme.point }}>{calculateDDay()}</div>
                                            <div className="h-2.5 w-[1px] bg-blue-200 mx-0.5" />
                                            <input type="date" value={dDayTarget} onChange={(e) => handleDDayChange(e.target.value)} className="text-[9px] bg-transparent border-none p-0 focus:ring-0 outline-none font-bold text-blue-600 cursor-pointer w-24" />
                                        </div>
                                        <div className="flex gap-1 p-1 bg-blue-50/50 rounded-full border border-blue-100">
                                            <button onClick={() => setView('daily')} className={`px-4 py-1 text-[10px] font-black rounded-full transition-all ${view === 'daily' ? 'bg-white shadow-md' : 'text-blue-300'}`} style={view === 'daily' ? { color: theme.point } : {}}>PLANNER</button>
                                            <button onClick={() => setView('archive')} className={`px-4 py-1 text-[10px] font-black rounded-full transition-all ${view === 'archive' ? 'bg-white shadow-md' : 'text-blue-300'}`} style={view === 'archive' ? { color: theme.point } : {}}>ARCHIVE</button>
                                        </div>
                                    </div>
                                </div>
                                <div className="flex items-center gap-3">
                                    <span className="text-[8px] text-blue-200 font-mono">UID: {user?.uid?.substring(0,8)}...</span>
                                    <i data-lucide="settings" className="w-4 h-4 text-blue-200 cursor-pointer hover:rotate-45 transition-transform"></i>
                                </div>
                            </div>

                            {view === 'daily' ? (
                                <div className="flex-1 flex overflow-hidden">
                                    <div className="flex-[1.8] border-r border-blue-50 p-5 overflow-hidden flex flex-col">
                                        <div className="mb-4 p-3 rounded-xl border-2 flex items-center gap-3 shrink-0" style={{ backgroundColor: '#F0F9FF', borderColor: theme.primary }}>
                                           <div className="w-1.5 h-6 rounded-full" style={{ backgroundColor: theme.point }} />
                                           <textarea placeholder="오늘의 다짐 한 문장..." value={currentRecord.resolution} onChange={(e) => saveRecordToCloud({ resolution: e.target.value })} className="flex-1 bg-transparent text-[16px] font-black border-none focus:ring-0 resize-none h-6 leading-tight placeholder:text-blue-200 checklist-font text-gray-800" />
                                        </div>
                                        <div className="flex-1 overflow-y-auto scrollbar-hide">
                                            <div className="flex justify-between items-center mb-4 sticky top-0 bg-white z-10 pb-2">
                                                <span className="text-[11px] font-black text-white tracking-widest px-3 py-1 rounded-full shadow-md uppercase" style={{ backgroundColor: theme.point }}>Tasks</span>
                                                <button onClick={() => saveRecordToCloud({ todos: [...currentRecord.todos, { id: Date.now(), text: "", completed: false }] })} className="w-9 h-9 text-white rounded-full flex items-center justify-center shadow-lg hover:scale-110 active:scale-95" style={{ backgroundColor: theme.sidebar }}><i data-lucide="plus" className="w-7 h-7"></i></button>
                                            </div>
                                            <div className="space-y-2 px-1 pb-10">
                                                {currentRecord.todos.map((todo) => (
                                                    <div key={todo.id} className="flex items-center gap-4 group">
                                                        <button onClick={() => saveRecordToCloud({ todos: currentRecord.todos.map(t => t.id === todo.id ? { ...t, completed: !t.completed } : t) })} className="shrink-0 relative">
                                                            <div className={`w-8 h-8 rounded-lg border-2 transition-all ${todo.completed ? 'scale-90' : 'border-blue-100 bg-blue-50/30'}`} style={todo.completed ? { borderColor: theme.primary, backgroundColor: theme.primary } : {}} />
                                                            {todo.completed && <svg className="absolute inset-0 m-auto w-5 h-5 text-blue-800" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="5" d="M5 13l4 4L19 7" /></svg>}
                                                        </button>
                                                        <input type="text" value={todo.text} onChange={(e) => saveRecordToCloud({ todos: currentRecord.todos.map(t => t.id === todo.id ? { ...t, text: e.target.value } : t) })} placeholder="할 일을 입력하세요..." className={`flex-1 bg-transparent border-none focus:ring-0 text-[18px] p-1 checklist-font ${todo.completed ? 'text-blue-200 line-through italic' : 'text-gray-800'}`} />
                                                        <button onClick={() => saveRecordToCloud({ todos: currentRecord.todos.filter(t => t.id !== todo.id) })} className="opacity-0 group-hover:opacity-100 p-1.5 text-gray-200 hover:text-red-400 transition-opacity"><i data-lucide="trash-2" className="w-4 h-4"></i></button>
                                                    </div>
                                                ))}
                                            </div>
                                        </div>
                                    </div>

                                    <div className="flex-[1] p-5 flex flex-col gap-4">
                                        <div className="flex-1 flex flex-col min-h-0">
                                            <div className="flex items-center gap-2 mb-3">
                                                <i data-lucide="clock" className="w-3.5 h-3.5" style={{ color: theme.point }}></i>
                                                <h3 className="text-[10px] font-black text-blue-800 tracking-[0.2em] uppercase">Weekly Timetable</h3>
                                            </div>
                                            <div className="flex-1 bg-white border-2 border-blue-50 rounded-xl overflow-hidden flex flex-col shadow-sm">
                                                <div className="grid grid-cols-5 shrink-0" style={{ backgroundColor: theme.sidebar }}>
                                                    {["월", "화", "수", "목", "금"].map(day => <div key={day} className="py-1 text-center text-[9px] font-black text-blue-100 border-r border-white/10 uppercase last:border-r-0">{day}</div>)}
                                                </div>
                                                <div className="flex-1 grid grid-rows-7 overflow-hidden">
                                                    {timetable.map((row, r) => (
                                                        <div key={r} className="grid grid-cols-5 border-b border-blue-50 last:border-b-0">
                                                            {row.map((sub, c) => (
                                                                <div key={c} className="p-0.5 border-r border-blue-50 flex items-center justify-center text-center last:border-r-0 hover:bg-blue-50 transition-colors">
                                                                    <span className="font-bold text-gray-700 text-[13px] leading-tight line-clamp-2">{sub}</span>
                                                                </div>
                                                            ))}
                                                        </div>
                                                    ))}
                                                </div>
                                            </div>
                                        </div>
                                        
                                        <div className="shrink-0">
                                            <div className="rounded-2xl p-2 px-3 flex items-center justify-between shadow-xl border border-white/20" style={{ backgroundColor: theme.timerBg }}>
                                                <button onClick={toggleTimer} className={`w-11 h-11 rounded-full flex items-center justify-center transition-all ${isTimerRunning ? 'text-blue-900 shadow-[0_0_20px_rgba(186,230,253,0.6)]' : 'bg-blue-900/40 hover:bg-blue-800'}`} style={isTimerRunning ? { backgroundColor: theme.primary } : { color: theme.primary }}>
                                                    {isTimerRunning ? <i data-lucide="pause" className="w-5 h-5 fill-current"></i> : <i data-lucide="play" className="w-5 h-5 fill-current ml-0.5"></i>}
                                                </button>
                                                <div className="flex items-baseline gap-1">
                                                    <span className="text-2xl font-normal tabular-nums digital-font" style={{ color: theme.primary }}>
                                                        {String(Math.floor((seconds % 3600) / 60)).padStart(2, '0')}:{String(seconds % 60).padStart(2, '0')}
                                                    </span>
                                                    <span className="text-[10px] font-bold text-blue-300/80 digital-font">{Math.floor(seconds / 3600)}H</span>
                                                </div>
                                                <button onClick={resetTimer} className="w-11 h-11 bg-blue-900/40 rounded-full flex items-center justify-center text-blue-400 hover:text-blue-100 transition-colors">
                                                    <i data-lucide="rotate-ccw" className="w-5 h-5"></i>
                                                </button>
                                            </div>
                                        </div>
                                    </div>
                                </div>
                            ) : (
                                <div className="flex-1 p-8 overflow-y-auto bg-white scrollbar-hide">
                                    <h2 className="text-2xl font-black text-gray-900 italic mb-8 uppercase tracking-tighter">My History</h2>
                                    <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-4">
                                        {Object.keys(records).sort((a,b) => b.localeCompare(a)).map(date => (
                                            <div key={date} onClick={() => { setCurrentDate(date); setView('daily'); }} className="p-4 rounded-[32px] border-2 border-gray-50 bg-white shadow-md hover:scale-105 hover:border-blue-200 transition-all cursor-pointer">
                                                <div className="text-lg font-black text-gray-900 tracking-tighter">{date.substring(0,4)}/{date.substring(4,6)}/{date.substring(6,8)}</div>
                                                <div className="text-[10px] text-blue-400 truncate mb-3 italic">"{records[date].resolution || '내용 없음'}"</div>
                                                <div className="text-[10px] font-black pt-3 border-t border-blue-50" style={{ color: theme.point }}>
                                                    {Math.floor((records[date].totalTime || 0) / 3600)}H {String(Math.floor(((records[date].totalTime || 0) % 3600) / 60)).padStart(2, '0')}:{String((records[date].totalTime || 0) % 60).padStart(2, '0')}
                                                </div>
                                            </div>
                                        ))}
                                    </div>
                                </div>
                            )}
                        </div>
                    </div>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);

        // Initialize Lucide icons after rendering
        setInterval(() => { lucide.createIcons(); }, 1000);
    </script>
</body>
</html>
