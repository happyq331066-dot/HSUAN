import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { 
  getFirestore, doc, addDoc, setDoc, updateDoc, deleteDoc, 
  onSnapshot, collection, query, orderBy, getDocs
} from 'firebase/firestore';
import { 
  Calendar, Clock, User, CheckCircle, Plus, Trash2, Lock, LogOut, 
  ChevronLeft, ChevronRight, Phone, Info, Settings, X, MessageCircle, ExternalLink, Loader2,
  CalendarCheck, Ban, Check, AlertCircle, Edit, Search, Filter, Sparkles, Coffee, List, Store, ImageIcon, Key, Users, FileText, History, Briefcase, UserPlus, UserMinus, Moon, Sun, LayoutGrid, Star, Award, CalendarDays, ShieldAlert, Trophy, Wrench, BarChart3, TrendingUp, DollarSign, Percent, Coins, Calculator, Tag as TagIcon, RefreshCw, BadgeCheck, Eye, ShieldCheck, UserCog, CheckSquare, XSquare, FolderOpen, Save, Layers, Shield
} from 'lucide-react';

// --- 全局變數與 Firebase 設定 ---
// 處理環境變數，確保在本地或線上環境皆可運行
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-spa-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// --- 初始化 Firebase ---
// 注意：如果沒有配置 firebaseConfig，這些功能將無法連接後端，但應用程式仍可透過 LocalStorage 運行
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// --- 預設模擬數據 (支援分類結構) ---
const DEFAULT_MAIN_SERVICES = [
  { id: '2hr', name: '全身精油 SPA (2小時)', duration: 120, price: 2000, commission: null, category: '身體護理' },
  { id: '3hr', name: '全身精油 SPA (3小時)', duration: 180, price: 3000, commission: null, category: '身體護理' },
  { id: 'face1', name: '深層保濕護理', duration: 90, price: 1800, commission: null, category: '臉部保養' },
  { id: 'face2', name: '抗老緊緻拉提', duration: 100, price: 2500, commission: null, category: '臉部保養' },
];
const DEFAULT_PAID_ADDONS = [
  { id: 'mugwort', name: '艾草溫罐', duration: 40, price: 299, commission: null, category: '傳統調理' },
  { id: 'ear', name: '採耳', duration: 30, price: 500, commission: null, category: '舒壓放鬆' }
];
// 免費體驗改為物件結構以支援分類與時間
const DEFAULT_FREE_EXPERIENCES = [
  { id: 'f1', name: '刮痧', duration: 15, price: 0, category: '傳統調理' },
  { id: 'f2', name: '拔罐', duration: 15, price: 0, category: '傳統調理' },
  { id: 'f3', name: '滑罐', duration: 15, price: 0, category: '傳統調理' },
  { id: 'f4', name: '頭部舒壓', duration: 10, price: 0, category: '放鬆' },
  { id: 'f5', name: '耳燭 (20分)', duration: 20, price: 0, category: '放鬆' },
];
const DEFAULT_TIME_SLOTS = ['10:00', '11:00', '13:00', '14:00', '15:00', '16:00', '18:00', '19:00', '20:00'];

// 職位包含權限等級 (High, Medium, Low)
const DEFAULT_JOB_TITLES = [
    { name: '店長', level: 'high' },
    { name: '資深美甲師', level: 'medium' },
    { name: '美甲師', level: 'low' },
    { name: '助理', level: 'low' },
    { name: '芳療師', level: 'medium' },
    { name: '美容師', level: 'medium' }
];

const DEFAULT_SHOP_NAME = '舒心・精油 SPA';
const DEFAULT_ADMIN_USERNAME = 'admin'; 
const DEFAULT_ADMIN_PASSWORD = '8888'; 
const DEFAULT_RESERVATION_RULES = `1. 請準時抵達，預約保留 10 分鐘，逾時視同取消。
2. 如需更改或取消預約，請提前 24 小時告知。
3. 為了維護服務品質，請勿攜帶寵物。`;

// --- 顏色主題 ---
const THEME = {
  bg: 'bg-[#F9F7F2]', 
  primary: 'bg-[#C7B299]', 
  primaryHover: 'hover:bg-[#B59F85]',
  secondary: 'bg-[#EBE5DF]', 
  accent: 'text-[#8C7B6C]', 
  text: 'text-[#5E5049]', 
  border: 'border-[#E5D9CE]',
  card: 'bg-white',
  input: 'bg-[#FAF9F6]',
};

// --- 輔助函數：分組 ---
const groupBy = (array, key) => {
  if (!Array.isArray(array)) return {};
  return array.reduce((result, currentValue) => {
    const groupKey = currentValue[key] || '其他';
    if (!result[groupKey]) {
      result[groupKey] = [];
    }
    result[groupKey].push(currentValue);
    return result;
  }, {});
};

// --- 輔助組件 ---
const Card = ({ children, className = "", onClick }) => (
  <div onClick={onClick} className={`${THEME.card} rounded-2xl shadow-sm border ${THEME.border} overflow-hidden ${className}`}>
    {children}
  </div>
);

const Button = ({ children, onClick, variant = 'primary', className = "", disabled = false, loading = false, type="button" }) => {
  const baseStyle = "px-6 py-3 rounded-xl font-medium transition-all duration-300 flex items-center justify-center gap-2 active:scale-95 disabled:opacity-50 disabled:active:scale-100 disabled:cursor-not-allowed";
  const variants = {
    primary: `${THEME.primary} ${THEME.primaryHover} text-white shadow-md`,
    outline: `border-2 ${THEME.border} text-[#8C7B6C] hover:bg-[#F5F0EB]`,
    text: "text-[#8C7B6C] hover:bg-[#F5F0EB]",
    danger: "bg-red-50 text-red-400 hover:bg-red-100 border border-red-100",
    success: "bg-green-50 text-green-600 hover:bg-green-100 border border-green-100",
    sm: "px-3 py-1.5 text-sm"
  };
  
  let finalClass = variants[variant] || variants.primary;
  if (variant === 'danger-sm') finalClass = `${variants.danger} ${variants.sm}`;
  if (variant === 'primary-sm') finalClass = `${variants.primary} ${variants.sm}`;
  if (variant === 'success-sm') finalClass = `${variants.success} ${variants.sm}`;
  
  return (
    <button type={type} onClick={onClick} className={`${baseStyle} ${finalClass} ${className}`} disabled={disabled || loading}>
      {loading ? <Loader2 size={20} className="animate-spin" /> : children}
    </button>
  );
};

const Tag = ({ children, status }) => {
  let colorClass = 'bg-stone-100 text-stone-500';
  if (status === 'pending') colorClass = 'bg-yellow-50 text-yellow-700 border border-yellow-100';
  if (status === 'completed') colorClass = 'bg-green-50 text-green-700 border border-green-100';
  if (status === 'cancelled') colorClass = 'bg-red-50 text-red-700 border border-red-100';
  if (status === 'Gold') colorClass = 'bg-amber-100 text-amber-700 border border-amber-200';
  if (status === 'Silver') colorClass = 'bg-slate-100 text-slate-600 border border-slate-200';
  if (status === 'Bronze') colorClass = 'bg-orange-50 text-orange-700 border border-orange-100';
  
  return <span className={`px-2 py-0.5 rounded-lg text-xs font-bold ${colorClass}`}>{children}</span>;
};

// --- 主應用程式 ---
export default function App() {
  const [view, setView] = useState('booking'); 
  const [userId, setUserId] = useState(null);
  const [appointments, setAppointments] = useState([]);
  const [members, setMembers] = useState([]); 
  const [staff, setStaff] = useState([]); 
  const [roster, setRoster] = useState([]); 
  const [settings, setSettings] = useState({ 
    shopName: DEFAULT_SHOP_NAME,
    adminUsername: DEFAULT_ADMIN_USERNAME,
    adminPassword: DEFAULT_ADMIN_PASSWORD,
    timeSlots: DEFAULT_TIME_SLOTS, 
    mainServices: DEFAULT_MAIN_SERVICES,
    freeExperiences: DEFAULT_FREE_EXPERIENCES,
    paidAddons: DEFAULT_PAID_ADDONS,
    reservationRules: DEFAULT_RESERVATION_RULES,
    jobTitles: DEFAULT_JOB_TITLES 
  });
  const [currentUser, setCurrentUser] = useState(null); 

  useEffect(() => {
    const initApp = async () => {
        try {
            // 優先嘗試自定義 Token 登入，否則匿名登入 (符合 Rule 3)
            if (initialAuthToken) {
                await signInWithCustomToken(auth, initialAuthToken);
            } else {
                await signInAnonymously(auth);
            }

            // 本地資料加載邏輯
            const savedApps = JSON.parse(localStorage.getItem('spa_appointments')) || [];
            const savedMembers = JSON.parse(localStorage.getItem('spa_members')) || [];
            const savedStaff = JSON.parse(localStorage.getItem('spa_staff')) || [
                { id: 's1', name: 'Emily', username: 'emily', role: '店長', defaultStart: '10:00', defaultEnd: '19:00', commissionRate: 50 },
                { id: 's2', name: 'Jessica', username: 'jessica', role: '資深美甲師', defaultStart: '12:00', defaultEnd: '21:00', commissionRate: 60 },
                { id: 's3', name: 'Nana', username: 'nana', role: '美甲師', defaultStart: '10:00', defaultEnd: '18:00', commissionRate: 50 }
            ];
            const savedRoster = JSON.parse(localStorage.getItem('spa_roster')) || [];
            const savedSettings = JSON.parse(localStorage.getItem('spa_settings')) || settings;

            // 資料遷移
            let finalSettings = { ...settings, ...savedSettings };
            // 確保 Job Titles 是物件陣列
            if (finalSettings.jobTitles && typeof finalSettings.jobTitles[0] === 'string') {
                finalSettings.jobTitles = finalSettings.jobTitles.map(t => ({ name: t, level: t === '店長' ? 'high' : 'low' }));
            }
            // 確保 Free Experiences 是物件陣列
            if (finalSettings.freeExperiences.length > 0 && typeof finalSettings.freeExperiences[0] === 'string') {
                finalSettings.freeExperiences = finalSettings.freeExperiences.map((name, idx) => ({
                    id: `legacy_${idx}`, name, duration: 15, price: 0, category: '其他'
                }));
            }

            setAppointments(savedApps);
            setMembers(savedMembers);
            setStaff(savedStaff);
            setRoster(savedRoster);
            setSettings(finalSettings);
            
            setUserId(auth.currentUser?.uid || 'local-user');
        } catch (e) { 
            console.error("Initialization error:", e);
            setUserId('offline-user');
        }
    };
    initApp();
  }, []);

  const saveToLocal = (key, data) => {
      localStorage.setItem(key, JSON.stringify(data));
      return data;
  };

  const handleResetSystem = () => {
      if(confirm('確定要重置所有系統資料嗎？這將會清除所有訂單、會員與自訂設定，恢復為預設值。')) {
          localStorage.clear();
          window.location.reload();
      }
  };

  const handleBook = async (data) => {
      try {
        const newApp = { ...data, id: Date.now().toString(), createdAt: new Date().toISOString(), status: 'pending' };
        const newApps = [newApp, ...appointments];
        await new Promise(resolve => setTimeout(resolve, 800));
        setAppointments(saveToLocal('spa_appointments', newApps));
        
        const cleanPhone = data.phone.replace(/\D/g, '');
        const existingIdx = members.findIndex(m => m.phone.replace(/\D/g, '') === cleanPhone);
        let newMembers = [...members];
        
        if(existingIdx >= 0) {
            newMembers[existingIdx] = { 
                ...newMembers[existingIdx], 
                lastVisit: new Date().toISOString(), 
                name: data.name,
                lineId: data.lineId 
            };
        } else {
            newMembers.push({ 
                id: cleanPhone, name: data.name, phone: data.phone, lineId: data.lineId, 
                level: 'Bronze', points: 0, lastVisit: new Date().toISOString(), adminNotes: ''
            });
        }
        setMembers(saveToLocal('spa_members', newMembers));
        return 'success';
      } catch (e) { return 'error'; }
  };
  
  const addManualHistory = (data) => {
      const newApp = {
          ...data,
          id: `manual_${Date.now()}`,
          createdAt: new Date().toISOString(),
          status: 'completed',
          isManual: true, 
          time: '00:00' 
      };
      const newApps = [newApp, ...appointments];
      setAppointments(saveToLocal('spa_appointments', newApps));
      
      const cleanPhone = data.phone.replace(/\D/g, '');
      const existingIdx = members.findIndex(m => m.phone.replace(/\D/g, '') === cleanPhone);
      if(existingIdx >= 0) {
          const newMembers = [...members];
          newMembers[existingIdx] = { ...newMembers[existingIdx], lastVisit: data.date };
          setMembers(saveToLocal('spa_members', newMembers));
      }
  };

  const updateAppointment = (id, data) => {
      const newApps = appointments.map(a => a.id === id ? { ...a, ...data } : a);
      setAppointments(saveToLocal('spa_appointments', newApps));
  };

  const deleteAppointment = (id) => {
      const newApps = appointments.filter(a => a.id !== id);
      setAppointments(saveToLocal('spa_appointments', newApps));
  };

  const updateRoster = (date, staffId, data) => {
      const rosterId = `${date}_${staffId}`;
      const existingIdx = roster.findIndex(r => r.id === rosterId);
      let newRoster = [...roster];
      if (existingIdx >= 0) {
          newRoster[existingIdx] = { ...newRoster[existingIdx], ...data };
      } else {
          newRoster.push({ id: rosterId, date, staffId, ...data });
      }
      setRoster(saveToLocal('spa_roster', newRoster));
  };

  const addStaff = (data) => {
      const newS = { ...data, id: `staff_${Date.now()}` };
      setStaff(saveToLocal('spa_staff', [...staff, newS]));
  };
  const deleteStaff = (id) => {
      setStaff(saveToLocal('spa_staff', staff.filter(s => s.id !== id)));
  };
  
  const updateStaff = (id, data) => {
      const newStaffList = staff.map(s => s.id === id ? { ...s, ...data } : s);
      setStaff(saveToLocal('spa_staff', newStaffList));
  };

  const addMember = (data) => {
      const cleanPhone = data.phone.replace(/\D/g, '');
      if (!cleanPhone) return alert('請輸入有效電話');
      const exists = members.find(m => m.phone.replace(/\D/g, '') === cleanPhone);
      if (exists) return alert('此會員電話已存在');
      
      const newMember = {
          ...data,
          id: cleanPhone,
          lastVisit: '', 
      };
      setMembers(saveToLocal('spa_members', [...members, newMember]));
  };

  const updateMember = (id, data) => {
      const newMembers = members.map(m => m.id === id ? { ...m, ...data } : m);
      setMembers(saveToLocal('spa_members', newMembers));
  };

  const updateMemberPoints = (id, points) => {
      const newMembers = members.map(m => m.id === id ? { ...m, points } : m);
      setMembers(saveToLocal('spa_members', newMembers));
  };

  const updateSettings = (newSettings) => {
      setSettings(saveToLocal('spa_settings', { ...settings, ...newSettings }));
  };

  if (!userId) return <div className="min-h-screen flex items-center justify-center bg-[#F9F7F2]"><Loader2 size={48} className="animate-spin text-[#C7B299]" /></div>;

  return (
    <div className={`min-h-screen ${THEME.bg} ${THEME.text} font-sans pb-20`}>
      {view === 'booking' && (
        <nav className="bg-white/80 backdrop-blur-md sticky top-0 z-50 border-b border-[#E5D9CE] px-4 py-4">
            <div className="max-w-md mx-auto flex justify-between items-center">
            <div className="flex items-center gap-2 font-bold text-lg tracking-wide cursor-pointer" onClick={() => setView('booking')}>
                <div className="w-8 h-8 rounded-full bg-[#C7B299] text-white flex items-center justify-center">{settings.shopName[0]}</div>
                {settings.shopName}
            </div>
            <button onClick={() => setView('admin_login')} className="p-2 rounded-full hover:bg-[#EBE5DF] transition-colors">
                <User size={20} />
            </button>
            </div>
        </nav>
      )}

      <main className="max-w-md mx-auto">
        {view === 'booking' && <BookingFlow onBook={handleBook} settings={settings} staff={staff} roster={roster} members={members} />}
        {view === 'admin_login' && (
            <AdminLogin 
                onSuccess={(user) => { setCurrentUser(user); setView('admin_panel'); }} 
                settings={settings} 
                staffList={staff}
                onBack={() => setView('booking')}
            />
        )}
        {view === 'admin_panel' && (
          <AdminPanel 
            appointments={appointments} 
            members={members}
            staff={staff}
            roster={roster}
            currentUser={currentUser}
            updateRoster={updateRoster}
            addStaff={addStaff}
            deleteStaff={deleteStaff}
            updateStaff={updateStaff} 
            addMember={addMember}
            updateMemberPoints={updateMemberPoints}
            updateMember={updateMember}
            updateAppointment={updateAppointment}
            deleteAppointment={deleteAppointment}
            updateSettings={updateSettings}
            onLogout={() => { setCurrentUser(null); setView('booking'); }}
            settings={settings}
            onResetSystem={handleResetSystem}
            handleBook={handleBook}
            addManualHistory={addManualHistory}
          />
        )}
      </main>
    </div>
  );
}

// --- 組件定義 ---

function AdminLogin({ onSuccess, settings, staffList, onBack }) {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const [error, setError] = useState('');

    const handleLogin = () => {
        if (username === settings.adminUsername && password === settings.adminPassword) {
            onSuccess({ name: '店長', role: 'admin', permissionLevel: 'high' });
        } else {
            const staff = staffList.find(s => s.username === username && s.password === password);
            if (staff) {
                const jobTitleConfig = settings.jobTitles.find(t => t.name === staff.role);
                const level = jobTitleConfig ? jobTitleConfig.level : 'low';
                onSuccess({ ...staff, permissionLevel: level }); 
            } else {
                setError('帳號或密碼錯誤');
            }
        }
    };

    return (
        <div className="min-h-screen flex flex-col items-center justify-center p-8 bg-[#FDFBF7]">
            <div className="w-full max-w-sm space-y-8">
                <div className="text-center space-y-4">
                    <div className="w-20 h-20 bg-[#D4C3B3] rounded-[2rem] flex items-center justify-center mx-auto shadow-lg text-white">
                        <Lock size={36} />
                    </div>
                    <div><h1 className="text-2xl font-bold text-[#5E5049] tracking-wide">店務管理系統</h1><p className="text-[#8C7B6C] text-sm mt-2">請輸入人員姓名與密碼</p></div>
                </div>
                <div className="space-y-6 mt-8">
                    <div className="space-y-2"><label className="text-sm font-bold text-[#5E5049] ml-1">帳號 (姓名)</label><input type="text" className="w-full p-4 bg-white border border-[#E5D9CE] rounded-2xl outline-none focus:border-[#C7B299] focus:ring-2 focus:ring-[#EBE5DF] transition-all text-[#5E5049]" value={username} onChange={e => setUsername(e.target.value)}/></div>
                    <div className="space-y-2"><label className="text-sm font-bold text-[#5E5049] ml-1">密碼</label><input type="password" className="w-full p-4 bg-white border border-[#E5D9CE] rounded-2xl outline-none focus:border-[#C7B299] focus:ring-2 focus:ring-[#EBE5DF] transition-all text-[#5E5049]" value={password} onChange={e => setPassword(e.target.value)}/></div>
                    {error && <p className="text-red-400 text-center text-sm">{error}</p>}
                    <button onClick={handleLogin} className="w-full py-4 bg-[#C7B299] hover:bg-[#B59F85] text-white rounded-2xl font-bold text-lg shadow-lg shadow-[#EBE5DF] transition-all active:scale-95">登入系統</button>
                    <button onClick={onBack} className="w-full text-[#8C7B6C] text-sm hover:text-[#5E5049]">返回前台</button>
                </div>
            </div>
        </div>
    );
}

function AdminPanel(props) {
    const { currentUser, onLogout, updateAppointment, updateMember, addMember, updateStaff, deleteStaff, settings, staff, onResetSystem, appointments, handleBook, members, addManualHistory } = props; 
    const [tab, setTab] = useState('calendar'); 
    const [editingApp, setEditingApp] = useState(null); 
    const [editingMember, setEditingMember] = useState(null);
    const [editingStaff, setEditingStaff] = useState(null); 
    const [viewHistoryMember, setViewHistoryMember] = useState(null); 
    const [isCreating, setIsCreating] = useState(false); 
    const [isAddingMember, setIsAddingMember] = useState(false); 

    const level = currentUser?.permissionLevel || 'low';
    const isHigh = level === 'high';
    const isMedium = level === 'medium' || level === 'high';

    const TabBtn = ({ id, label, icon: Icon }) => (
        <button onClick={() => setTab(id)} className={`flex-1 min-w-[64px] py-3 text-xs font-bold transition-all rounded-xl flex flex-col items-center gap-1 ${tab === id ? 'bg-[#C7B299] text-white shadow-md' : 'text-[#8C7B6C] hover:bg-[#EBE5DF]'}`}>
            {Icon && <Icon size={18} />}{label}
        </button>
    );

    return (
        <div className="min-h-screen bg-[#FDFBF7] pb-20">
            <header className="bg-[#FDFBF7] p-4 sticky top-0 z-10 shadow-sm">
                <div className="flex justify-between items-center mb-4">
                    <h2 className="text-xl font-bold text-[#5E5049] flex items-center gap-2"><LayoutGrid size={20} /> {currentUser?.name} <span className="text-xs bg-[#EBE5DF] px-2 py-0.5 rounded-full text-[#8C7B6C]">{settings.jobTitles.find(j=>j.name===currentUser.role)?.name || currentUser.role}</span></h2>
                    <div className="flex gap-2">
                        <button onClick={() => setIsCreating(true)} className="flex items-center gap-1 text-sm bg-[#C7B299] text-white px-3 py-1.5 rounded-lg shadow-sm hover:bg-[#B59F85] transition-colors"><Plus size={16} /> 新增預約</button>
                        <button onClick={onLogout} className="w-10 h-10 rounded-full bg-[#EBE5DF] flex items-center justify-center text-[#8C7B6C]"><LogOut size={18} /></button>
                    </div>
                </div>
                <div className="flex p-1 bg-white border border-[#E5D9CE] rounded-2xl shadow-sm gap-1 overflow-x-auto no-scrollbar">
                    <TabBtn id="calendar" label="行事曆" icon={CalendarDays} />
                    <TabBtn id="roster" label="排班" icon={Calendar} />
                    <TabBtn id="booking" label="列表" icon={List} />
                    <TabBtn id="members" label="會員" icon={Users} />
                    {isMedium && <TabBtn id="performance" label="業績" icon={BarChart3} />}
                    {isHigh && <TabBtn id="staff" label="人員" icon={Briefcase} />}
                    {isHigh && <TabBtn id="settings" label="設定" icon={Settings} />}
                </div>
            </header>

            <div className="p-4">
                {tab === 'calendar' && <AdminCalendarView {...props} onEdit={setEditingApp} />}
                {tab === 'roster' && <RosterView {...props} />}
                {tab === 'booking' && <BookingListView {...props} onEdit={setEditingApp} />}
                {tab === 'members' && <MemberListView {...props} onEdit={setEditingMember} onViewHistory={setViewHistoryMember} onAdd={() => setIsAddingMember(true)} />}
                {tab === 'performance' && isMedium && <PerformanceView {...props} />}
                {tab === 'staff' && isHigh && <StaffListView {...props} onEdit={setEditingStaff} isAdmin={isHigh} />}
                {tab === 'settings' && isHigh && <SettingsView {...props} onResetSystem={onResetSystem} currentUser={currentUser} />}
            </div>

            {/* Modals */}
            {editingApp && (<EditAppointmentModal appointment={editingApp} settings={settings} staff={staff} members={members} onClose={() => setEditingApp(null)} onSave={(id, data) => { updateAppointment(id, data); setEditingApp(null); }} />)}
            {isCreating && (<EditAppointmentModal appointment={{ date: new Date().toISOString().split('T')[0], time: '10:00', name: '', phone: '', status: 'pending', staff: null, mainService: settings.mainServices[0] }} settings={settings} staff={staff} members={members} onClose={() => setIsCreating(false)} onSave={async (id, data) => { await handleBook(data); setIsCreating(false); }} />)}
            {editingMember && (<EditMemberModal member={editingMember} staff={staff} onClose={() => setEditingMember(null)} onSave={(id, data) => { updateMember(id, data); setEditingMember(null); }} />)}
            {isAddingMember && (<EditMemberModal member={{ name: '', phone: '', lineId: '', level: 'Bronze', points: 0, adminNotes: '', preferredStaffId: '' }} staff={staff} onClose={() => setIsAddingMember(false)} onSave={(id, data) => { addMember(data); setIsAddingMember(false); }} />)}
            {viewHistoryMember && (<MemberHistoryModal member={viewHistoryMember} appointments={appointments} onClose={() => setViewHistoryMember(null)} onAddRecord={addManualHistory} settings={settings} staff={staff} />)}
            {editingStaff && (
                <div className="fixed inset-0 bg-black/40 backdrop-blur-sm z-50 flex items-end sm:items-center justify-center p-0 sm:p-4">
                    <div className="bg-white w-full max-w-lg rounded-t-3xl sm:rounded-3xl shadow-2xl p-6 space-y-4 max-h-[90vh] overflow-y-auto custom-scrollbar animate-fadeIn">
                        <div className="flex justify-between items-center border-b border-[#E5D9CE] pb-3"><h3 className="font-bold text-lg text-[#5E5049]">編輯人員</h3><button onClick={() => setEditingStaff(null)}><X /></button></div>
                        <div className="grid grid-cols-2 gap-4">
                             <div><label className="text-xs font-bold text-[#8C7B6C]">姓名</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.name} onChange={e => setEditingStaff({...editingStaff, name: e.target.value})} /></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">職稱</label><select className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.role} onChange={e => setEditingStaff({...editingStaff, role: e.target.value})}>{settings.jobTitles.map(r => <option key={r.name} value={r.name}>{r.name}</option>)}</select></div>
                        </div>
                        <div className="grid grid-cols-2 gap-4">
                             <div><label className="text-xs font-bold text-[#8C7B6C]">帳號</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.username} onChange={e => setEditingStaff({...editingStaff, username: e.target.value})} /></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">密碼</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.password} onChange={e => setEditingStaff({...editingStaff, password: e.target.value})} /></div>
                        </div>
                        <div className="grid grid-cols-2 gap-4">
                            <div><label className="text-xs font-bold text-[#8C7B6C]">預設早班</label><input type="time" className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.defaultStart || '10:00'} onChange={e => setEditingStaff({...editingStaff, defaultStart: e.target.value})} /></div>
                            <div><label className="text-xs font-bold text-[#8C7B6C]">預設晚班</label><input type="time" className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.defaultEnd || '19:00'} onChange={e => setEditingStaff({...editingStaff, defaultEnd: e.target.value})} /></div>
                        </div>
                        {isHigh && (<div><label className="text-xs font-bold text-[#8C7B6C]">個人抽成 (%)</label><input type="number" className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={editingStaff.commissionRate || 0} onChange={e => setEditingStaff({...editingStaff, commissionRate: e.target.value})} /></div>)}
                        <div className="pt-2 border-t border-[#E5D9CE]"><Button onClick={() => { updateStaff(editingStaff.id, editingStaff); setEditingStaff(null); }}>儲存變更</Button></div>
                    </div>
                </div>
            )}
        </div>
    );
}

function PerformanceView({ appointments, staff, settings, updateSettings, updateStaff }) {
    const [date, setDate] = useState(new Date().toISOString().split('T')[0]); 
    const [month, setMonth] = useState(new Date().toISOString().slice(0, 7)); 

    const calculateCommission = (app, staffMember) => {
        if (app.manualCommission !== undefined && app.manualCommission !== null && app.manualCommission !== '') {
            return Number(app.manualCommission);
        }
        let totalCommission = 0;
        if (app.mainService) {
            const serviceRate = app.mainService.commission ? Number(app.mainService.commission) : Number(staffMember.commissionRate || 0);
            totalCommission += Number(app.mainService.price) * (serviceRate / 100);
        }
        if (app.addOns) {
            app.addOns.forEach(addon => {
                const addonRate = addon.commission ? Number(addon.commission) : Number(staffMember.commissionRate || 0);
                totalCommission += Number(addon.price) * (addonRate / 100);
            });
        }
        return Math.round(totalCommission);
    };

    const calcStats = (apps, staffList) => {
        return staffList.map(s => {
            const staffApps = apps.filter(a => a.staff?.id === s.id);
            const revenue = staffApps.reduce((sum, a) => sum + Number(a.totalPrice), 0);
            const commission = staffApps.reduce((sum, a) => sum + calculateCommission(a, s), 0);
            return { name: s.name, revenue, commission, count: staffApps.length };
        }).sort((a, b) => b.revenue - a.revenue);
    };

    const dailyApps = appointments.filter(a => a.status === 'completed' && a.date === date);
    const dailyStats = calcStats(dailyApps, staff);
    const dailyTotal = dailyStats.reduce((acc, cur) => ({ rev: acc.rev + cur.revenue, com: acc.com + cur.commission }), { rev: 0, com: 0 });

    const monthlyApps = appointments.filter(a => a.status === 'completed' && a.date.startsWith(month));
    const monthlyStats = calcStats(monthlyApps, staff);
    const monthlyTotal = monthlyStats.reduce((acc, cur) => ({ rev: acc.rev + cur.revenue, com: acc.com + cur.commission }), { rev: 0, com: 0 });

    const updateServiceCommission = (id, comm, type) => {
        if (type === 'main') {
            const newServices = settings.mainServices.map(s => s.id === id ? { ...s, commission: comm } : s);
            updateSettings({ mainServices: newServices });
        } else {
            const newAddons = settings.paidAddons.map(s => s.id === id ? { ...s, commission: comm } : s);
            updateSettings({ paidAddons: newAddons });
        }
    };

    return (
        <div className="space-y-6 animate-fadeIn">
            <Card className="p-5">
                <h3 className="font-bold text-[#5E5049] mb-4 flex items-center gap-2"><TagIcon size={18}/> 服務項目抽成設定 (若未設定則依人員%)</h3>
                <div className="space-y-2 max-h-60 overflow-y-auto custom-scrollbar pr-2">
                    {settings.mainServices.map(s => (
                        <div key={s.id} className="flex justify-between items-center bg-[#F9F7F2] p-2 rounded-lg text-sm">
                            <span className="text-[#5E5049]">{s.name} (${s.price})</span>
                            <div className="flex items-center gap-1">
                                <input type="number" placeholder="預設" className="w-12 p-1 text-center border rounded text-xs" value={s.commission || ''} onChange={e => updateServiceCommission(s.id, e.target.value, 'main')} />
                                <span className="text-xs text-[#8C7B6C]">%</span>
                            </div>
                        </div>
                    ))}
                    {settings.paidAddons.map(s => (
                        <div key={s.id} className="flex justify-between items-center bg-[#F9F7F2] p-2 rounded-lg text-sm">
                            <span className="text-[#5E5049]">{s.name} (${s.price})</span>
                            <div className="flex items-center gap-1">
                                <input type="number" placeholder="預設" className="w-12 p-1 text-center border rounded text-xs" value={s.commission || ''} onChange={e => updateServiceCommission(s.id, e.target.value, 'addon')} />
                                <span className="text-xs text-[#8C7B6C]">%</span>
                            </div>
                        </div>
                    ))}
                </div>
                <p className="text-[10px] text-[#8C7B6C] mt-2">* 若未設定，將採用員工個人抽成比例。</p>
            </Card>

            <Card className="p-5">
                <div className="flex justify-between items-center mb-4 border-b border-[#E5D9CE] pb-3">
                    <div className="flex items-center gap-2"><TrendingUp className="text-[#C7B299]" size={20}/><h3 className="font-bold text-[#5E5049]">每日業績</h3></div>
                    <input type="date" className="p-1 bg-[#F9F7F2] border border-[#E5D9CE] rounded-lg text-xs text-[#5E5049]" value={date} onChange={e => setDate(e.target.value)} />
                </div>
                <div className="space-y-3">
                    {dailyStats.map(s => (
                        <div key={s.name} className="flex justify-between items-center text-sm">
                            <div className="text-[#5E5049] font-medium">{s.name} <span className="text-xs text-[#8C7B6C]">({s.count} 筆)</span></div>
                            <div className="text-right"><div className="font-bold text-[#C7B299]">${s.revenue.toLocaleString()}</div><div className="text-xs text-[#8C7B6C]">薪資: ${s.commission.toLocaleString()}</div></div>
                        </div>
                    ))}
                    <div className="pt-3 border-t border-[#E5D9CE] flex justify-between font-bold text-[#5E5049]"><span>總計</span><div className="text-right"><div>${dailyTotal.rev.toLocaleString()}</div><div className="text-xs text-[#8C7B6C]">總薪: ${dailyTotal.com.toLocaleString()}</div></div></div>
                </div>
            </Card>

            <Card className="p-5">
                <div className="flex justify-between items-center mb-4 border-b border-[#E5D9CE] pb-3">
                    <div className="flex items-center gap-2"><DollarSign className="text-[#C7B299]" size={20}/><h3 className="font-bold text-[#5E5049]">本月業績</h3></div>
                    <input type="month" className="p-1 bg-[#F9F7F2] border border-[#E5D9CE] rounded-lg text-xs text-[#5E5049]" value={month} onChange={e => setMonth(e.target.value)} />
                </div>
                <div className="space-y-3">
                    {monthlyStats.map((s, idx) => (
                        <div key={s.name} className="flex justify-between items-center text-sm">
                            <div className="flex items-center gap-2"><span className={`w-5 h-5 flex items-center justify-center rounded-full text-[10px] text-white ${idx < 3 ? 'bg-[#C7B299]' : 'bg-[#E5D9CE]'}`}>{idx + 1}</span><span className="text-[#5E5049] font-medium">{s.name}</span></div>
                            <div className="text-right"><div className="font-bold text-[#8B6F56]">${s.revenue.toLocaleString()}</div><div className="text-xs text-[#8C7B6C]">薪資: ${s.commission.toLocaleString()}</div></div>
                        </div>
                    ))}
                    <div className="pt-3 border-t border-[#E5D9CE] flex justify-between font-bold text-[#5E5049]"><span>總計</span><div className="text-right"><div>${monthlyTotal.rev.toLocaleString()}</div><div className="text-xs text-[#8C7B6C]">總薪: ${monthlyTotal.com.toLocaleString()}</div></div></div>
                </div>
            </Card>
        </div>
    );
}

function SettingsView({ settings, updateSettings, onResetSystem, currentUser }) {
    const [subTab, setSubTab] = useState('basic'); 
    
    // 新增/編輯暫存狀態
    const [newService, setNewService] = useState({ name: '', price: '', duration: '', category: '身體護理' });
    const [newAddon, setNewAddon] = useState({ name: '', price: '', duration: '', category: '傳統調理' }); 
    const [newFreeExp, setNewFreeExp] = useState({ name: '', duration: '15', price: 0, category: '傳統調理' }); 
    
    const [newTimeSlot, setNewTimeSlot] = useState('');
    const [newJobTitle, setNewJobTitle] = useState({ name: '', level: 'low' }); 
    
    const [adminForm, setAdminForm] = useState({ u: settings.adminUsername, p: settings.adminPassword });
    const [reservationRules, setReservationRules] = useState(settings.reservationRules || '');

    // 編輯狀態
    const [editingId, setEditingId] = useState(null);
    const [editForm, setEditForm] = useState({});

    const getCategories = (items) => [...new Set(items.map(i => i.category || '其他'))];

    // --- 服務項目操作 ---
    const addService = () => { 
        if(!newService.name) return; 
        updateSettings({ mainServices: [...settings.mainServices, { ...newService, id: Date.now().toString(), commission: null }] }); 
        setNewService({ name: '', price: '', duration: '', category: '身體護理' }); 
    };
    const removeService = (id) => updateSettings({ mainServices: settings.mainServices.filter(s => s.id !== id) });
    
    const addAddon = () => { 
        if(!newAddon.name) return; 
        updateSettings({ paidAddons: [...settings.paidAddons, { ...newAddon, id: `addon_${Date.now()}`, commission: null }] }); 
        setNewAddon({ name: '', price: '', duration: '', category: '傳統調理' }); 
    };
    const removeAddon = (id) => updateSettings({ paidAddons: settings.paidAddons.filter(s => s.id !== id) });
    
    const addFreeExp = () => { 
        if(!newFreeExp.name) return; 
        updateSettings({ freeExperiences: [...settings.freeExperiences, { ...newFreeExp, id: `free_${Date.now()}` }] }); 
        setNewFreeExp({ name: '', duration: '15', price: 0, category: '傳統調理' }); 
    };
    const removeFreeExp = (id) => updateSettings({ freeExperiences: settings.freeExperiences.filter(x => x.id !== id) });
    
    // --- 職稱操作 ---
    const addJobTitle = () => { 
        if(!newJobTitle.name) return; 
        updateSettings({ jobTitles: [...settings.jobTitles, newJobTitle] }); 
        setNewJobTitle({ name: '', level: 'low' }); 
    };
    const removeJobTitle = (name) => updateSettings({ jobTitles: settings.jobTitles.filter(t => t.name !== name) });

    // --- 編輯功能 ---
    const startEdit = (item) => {
        setEditingId(item.id);
        setEditForm({ ...item });
    };

    const cancelEdit = () => {
        setEditingId(null);
        setEditForm({});
    };

    const saveEdit = (type) => {
        if (type === 'main') {
            const newServices = settings.mainServices.map(s => s.id === editingId ? editForm : s);
            updateSettings({ mainServices: newServices });
        } else if (type === 'addon') {
            const newAddons = settings.paidAddons.map(s => s.id === editingId ? editForm : s);
            updateSettings({ paidAddons: newAddons });
        } else if (type === 'free') {
            const newFree = settings.freeExperiences.map(s => s.id === editingId ? editForm : s);
            updateSettings({ freeExperiences: newFree });
        }
        setEditingId(null);
    };

    // --- 分類選擇器組件 ---
    const CategorySelector = ({ items, current, onChange }) => {
        const existingCats = getCategories(items);
        const [isNew, setIsNew] = useState(false);
        const isCustom = current && !existingCats.includes(current) && current !== '__NEW__';

        return (
            <div className="col-span-1">
                 {!isNew && !isCustom ? (
                     <select className="w-full p-2 border rounded-lg text-sm bg-white" value={current} onChange={e => {
                         if(e.target.value === '__NEW__') setIsNew(true);
                         else onChange(e.target.value);
                     }}>
                         {existingCats.map(c => <option key={c} value={c}>{c}</option>)}
                         <option value="__NEW__">+ 新增分類...</option>
                     </select>
                 ) : (
                     <div className="flex gap-1">
                         <input autoFocus className="w-full p-2 border rounded-lg text-sm" placeholder="輸入分類" value={current === '__NEW__' ? '' : current} onChange={e => onChange(e.target.value)} />
                         <button onClick={() => { 
                             setIsNew(false); 
                             onChange(existingCats[0] || ''); 
                         }} className="text-[#8C7B6C]"><X size={14}/></button>
                     </div>
                 )}
            </div>
        );
    };

    return (
        <div className="space-y-6 animate-fadeIn">
             <div className="flex p-1 bg-white border border-[#E5D9CE] rounded-xl shadow-sm">
                <button onClick={()=>setSubTab('basic')} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all ${subTab==='basic' ? 'bg-[#C7B299] text-white shadow' : 'text-[#8C7B6C]'}`}>基本設定</button>
                <button onClick={()=>setSubTab('services')} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all ${subTab==='services' ? 'bg-[#C7B299] text-white shadow' : 'text-[#8C7B6C]'}`}>服務項目</button>
                <button onClick={()=>setSubTab('times')} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all ${subTab==='times' ? 'bg-[#C7B299] text-white shadow' : 'text-[#8C7B6C]'}`}>營業時段</button>
            </div>

            {subTab === 'basic' && (
                <>
                    <Card className="p-5 bg-[#FDFBF7] border-l-4 border-[#C7B299]">
                        <div className="flex items-center gap-2 mb-2 text-[#C7B299]">
                             <ShieldCheck size={20} />
                             <h3 className="font-bold">當前權限狀態</h3>
                        </div>
                        <div className="text-sm text-[#5E5049] space-y-1">
                             <p>登入身分：<span className="font-bold">{currentUser?.name}</span> ({currentUser?.role})</p>
                             <p>權限等級：
                                 {currentUser?.permissionLevel === 'high' ? <span className="bg-red-100 text-red-600 px-2 py-0.5 rounded text-xs ml-1">最高權限 (High)</span> : 
                                  currentUser?.permissionLevel === 'medium' ? <span className="bg-blue-100 text-blue-600 px-2 py-0.5 rounded text-xs ml-1">管理權限 (Medium)</span> : 
                                  <span className="bg-gray-100 text-gray-600 px-2 py-0.5 rounded text-xs ml-1">一般權限 (Low)</span>}
                             </p>
                        </div>
                    </Card>

                    <Card className="p-5 space-y-4">
                        <h3 className="font-bold text-[#5E5049] flex gap-2"><Store size={18} /> 店家與帳號</h3>
                        <div className="space-y-1"><label className="text-xs text-[#8C7B6C]">店家名稱</label><input className="w-full p-3 border border-[#E5D9CE] rounded-xl outline-none" value={settings.shopName} onChange={e => updateSettings({shopName: e.target.value})} placeholder="店名" /></div>
                        <div className="space-y-1"><label className="text-xs text-[#8C7B6C]">Logo 圖片連結</label><input className="w-full p-3 border border-[#E5D9CE] rounded-xl outline-none" value={settings.shopAvatar || ''} onChange={e => updateSettings({shopAvatar: e.target.value})} placeholder="https://..." /></div>
                        <div className="grid grid-cols-2 gap-3 pt-2 border-t border-[#E5D9CE]"><div><label className="text-xs text-[#8C7B6C]">管理員帳號</label><input className="w-full p-3 border border-[#E5D9CE] rounded-xl outline-none" value={adminForm.u} onChange={e => setAdminForm({...adminForm, u: e.target.value})} /></div><div><label className="text-xs text-[#8C7B6C]">管理員密碼</label><input className="w-full p-3 border border-[#E5D9CE] rounded-xl outline-none" value={adminForm.p} onChange={e => setAdminForm({...adminForm, p: e.target.value})} /></div></div>
                        <Button onClick={() => updateSettings({ adminUsername: adminForm.u, adminPassword: adminForm.p })}>更新帳密</Button>
                    </Card>
                    
                    <Card className="p-5 space-y-4">
                        <h3 className="font-bold text-[#5E5049] flex gap-2"><BadgeCheck size={18} /> 職位與權限設定</h3>
                        <div className="bg-[#F5F0EB] p-3 rounded-lg text-xs text-[#5E5049] space-y-1">
                            <p className="font-bold mb-1 border-b border-[#E5D9CE] pb-1">權限說明：</p>
                            <div className="grid grid-cols-[80px_1fr] gap-1">
                                <span className="font-bold text-red-600">最高權限</span>
                                <span>完整功能 (含業績、人員薪資、系統設定)。</span>
                                <span className="font-bold text-blue-600">管理權限</span>
                                <span>可看業績、排班、會員、預約，不可動系統設定。</span>
                                <span className="font-bold text-gray-600">一般權限</span>
                                <span>僅操作預約、會員、查看自身排班。</span>
                            </div>
                        </div>
                        <div className="space-y-2">
                            {settings.jobTitles.map((t, idx) => (
                                <div key={idx} className="flex items-center justify-between bg-[#F9F7F2] p-2 rounded-lg border border-[#E5D9CE]">
                                    <span className="font-bold text-[#5E5049]">{t.name}</span>
                                    <div className="flex items-center gap-2">
                                        <select 
                                            className={`text-xs p-1 rounded border outline-none ${t.level==='high'?'bg-red-50 text-red-600 border-red-200':t.level==='medium'?'bg-blue-50 text-blue-600 border-blue-200':'bg-gray-50 text-gray-600 border-gray-200'}`}
                                            value={t.level}
                                            onChange={(e) => {
                                                const newTitles = [...settings.jobTitles];
                                                newTitles[idx].level = e.target.value;
                                                updateSettings({ jobTitles: newTitles });
                                            }}
                                        >
                                            <option value="high">最高權限</option>
                                            <option value="medium">管理權限</option>
                                            <option value="low">一般權限</option>
                                        </select>
                                        <button onClick={() => removeJobTitle(t.name)} className="text-red-300 hover:text-red-500"><X size={16}/></button>
                                    </div>
                                </div>
                            ))}
                        </div>
                        <div className="flex gap-2 border-t border-[#E5D9CE] pt-3">
                            <input className="flex-1 p-2 border rounded-lg text-sm" placeholder="輸入新職稱" value={newJobTitle.name} onChange={e => setNewJobTitle({...newJobTitle, name: e.target.value})} />
                            <select className="p-2 border rounded-lg text-sm bg-white" value={newJobTitle.level} onChange={e => setNewJobTitle({...newJobTitle, level: e.target.value})}>
                                <option value="low">一般</option>
                                <option value="medium">管理</option>
                                <option value="high">最高</option>
                            </select>
                            <Button className="w-20" onClick={addJobTitle}>新增</Button>
                        </div>
                    </Card>

                    <Card className="p-5 space-y-4"><h3 className="font-bold text-[#5E5049] flex gap-2"><ShieldAlert size={18} /> 預約須知</h3><textarea className="w-full p-3 border border-[#E5D9CE] rounded-xl outline-none h-32 resize-none text-sm" value={reservationRules} onChange={(e) => setReservationRules(e.target.value)} /><Button onClick={() => { updateSettings({ reservationRules }); alert('預約須知已更新'); }}>儲存須知</Button></Card>
                </>
            )}

            {subTab === 'services' && (
                <>
                    {/* 主課程區塊 */}
                    <Card className="p-5 space-y-4">
                        <h3 className="font-bold text-[#5E5049] flex gap-2"><Sparkles size={18} /> 主課程</h3>
                        {Object.entries(groupBy(settings.mainServices, 'category')).map(([cat, items]) => (
                            <div key={cat} className="mb-4">
                                <h4 className="text-xs font-bold text-[#8C7B6C] mb-2 pl-1 border-l-2 border-[#C7B299]">{cat}</h4>
                                {items.map(s => (
                                    <div key={s.id} className="grid grid-cols-7 gap-2 items-center bg-[#F9F7F2] p-2 rounded-lg mb-2 text-sm">
                                        {editingId === s.id ? (
                                            <>
                                                <input className="col-span-2 p-1 border rounded" value={editForm.name} onChange={e=>setEditForm({...editForm, name: e.target.value})} />
                                                <input className="col-span-1 p-1 border rounded" value={editForm.price} onChange={e=>setEditForm({...editForm, price: e.target.value})} />
                                                <input className="col-span-1 p-1 border rounded" value={editForm.duration} onChange={e=>setEditForm({...editForm, duration: e.target.value})} />
                                                <div className="col-span-2"><CategorySelector items={settings.mainServices} current={editForm.category} onChange={val=>setEditForm({...editForm, category: val})} /></div>
                                                <div className="col-span-1 flex gap-1 justify-end">
                                                    <button onClick={() => saveEdit('main')} className="text-green-600"><Check size={16}/></button>
                                                    <button onClick={cancelEdit} className="text-red-400"><X size={16}/></button>
                                                </div>
                                            </>
                                        ) : (
                                            <>
                                                <div className="col-span-3 text-[#5E5049] truncate">{s.name}</div>
                                                <div className="col-span-1">${s.price}</div>
                                                <div className="col-span-1">{s.duration}分</div>
                                                <div className="col-span-2 text-right flex justify-end gap-2">
                                                    <button onClick={() => startEdit(s)} className="text-[#C7B299]"><Edit size={14}/></button>
                                                    <button onClick={() => removeService(s.id)} className="text-red-300"><Trash2 size={14}/></button>
                                                </div>
                                            </>
                                        )}
                                    </div>
                                ))}
                            </div>
                        ))}
                        
                        <div className="grid grid-cols-7 gap-2 mt-3 pt-3 border-t border-[#E5D9CE] bg-white">
                             <div className="col-span-7 text-xs font-bold text-[#8C7B6C] mb-1">新增主課程</div>
                            <input className="col-span-2 p-2 border rounded-lg text-sm" placeholder="名稱" value={newService.name} onChange={e => setNewService({...newService, name: e.target.value})} />
                            <input className="col-span-1 p-2 border rounded-lg text-sm" placeholder="$" value={newService.price} onChange={e => setNewService({...newService, price: e.target.value})} />
                            <input className="col-span-1 p-2 border rounded-lg text-sm" placeholder="分" value={newService.duration} onChange={e => setNewService({...newService, duration: e.target.value})} />
                            <div className="col-span-2"><CategorySelector items={settings.mainServices} current={newService.category} onChange={val => setNewService({...newService, category: val})} /></div>
                            <button onClick={addService} className="col-span-1 bg-[#C7B299] text-white rounded-lg flex items-center justify-center"><Plus size={18}/></button>
                        </div>
                    </Card>
                    
                    {/* 加購項目區塊 - 模式完全比照主課程 */}
                    <Card className="p-5 space-y-4">
                        <h3 className="font-bold text-[#5E5049] flex gap-2"><Coffee size={18} /> 加購項目</h3>
                        {Object.entries(groupBy(settings.paidAddons, 'category')).map(([cat, items]) => (
                            <div key={cat} className="mb-4">
                                <h4 className="text-xs font-bold text-[#8C7B6C] mb-2 pl-1 border-l-2 border-[#C7B299]">{cat}</h4>
                                {items.map(s => (
                                    <div key={s.id} className="grid grid-cols-7 gap-2 items-center bg-[#F9F7F2] p-2 rounded-lg mb-2 text-sm">
                                        {editingId === s.id ? (
                                            <>
                                                <input className="col-span-2 p-1 border rounded" value={editForm.name} onChange={e=>setEditForm({...editForm, name: e.target.value})} />
                                                <input className="col-span-1 p-1 border rounded" value={editForm.price} onChange={e=>setEditForm({...editForm, price: e.target.value})} />
                                                <input className="col-span-1 p-1 border rounded" value={editForm.duration} onChange={e=>setEditForm({...editForm, duration: e.target.value})} />
                                                <div className="col-span-2"><CategorySelector items={settings.paidAddons} current={editForm.category} onChange={val=>setEditForm({...editForm, category: val})} /></div>
                                                <div className="col-span-1 flex gap-1 justify-end">
                                                    <button onClick={() => saveEdit('addon')} className="text-green-600"><Check size={16}/></button>
                                                    <button onClick={cancelEdit} className="text-red-400"><X size={16}/></button>
                                                </div>
                                            </>
                                        ) : (
                                            <>
                                                <div className="col-span-3 text-[#5E5049] truncate">{s.name}</div>
                                                <div className="col-span-1">${s.price}</div>
                                                <div className="col-span-1">{s.duration}分</div>
                                                <div className="col-span-2 text-right flex justify-end gap-2">
                                                    <button onClick={() => startEdit(s)} className="text-[#C7B299]"><Edit size={14}/></button>
                                                    <button onClick={() => removeAddon(s.id)} className="text-red-300"><Trash2 size={14}/></button>
                                                </div>
                                            </>
                                        )}
                                    </div>
                                ))}
                            </div>
                        ))}
                        <div className="grid grid-cols-7 gap-2 mt-3 pt-3 border-t border-[#E5D9CE]">
                            <div className="col-span-7 text-xs font-bold text-[#8C7B6C] mb-1">新增加購項目</div>
                            <input className="col-span-2 p-2 border rounded-lg text-sm" placeholder="名稱" value={newAddon.name} onChange={e => setNewAddon({...newAddon, name: e.target.value})} />
                            <input className="col-span-1 p-2 border rounded-lg text-sm" placeholder="$" value={newAddon.price} onChange={e => setNewAddon({...newAddon, price: e.target.value})} />
                            <input className="col-span-1 p-2 border rounded-lg text-sm" placeholder="分" value={newAddon.duration} onChange={e => setNewAddon({...newAddon, duration: e.target.value})} />
                            <div className="col-span-2"><CategorySelector items={settings.paidAddons} current={newAddon.category} onChange={val => setNewAddon({...newAddon, category: val})} /></div>
                            <button onClick={addAddon} className="col-span-1 bg-[#C7B299] text-white rounded-lg flex items-center justify-center"><Plus size={18}/></button>
                        </div>
                    </Card>
                    
                    {/* 免費體驗區塊 - 模式完全比照主課程 (新增時間欄位) */}
                    <Card className="p-5 space-y-4">
                        <h3 className="font-bold text-[#5E5049] flex gap-2"><List size={18} /> 免費體驗</h3>
                        {Object.entries(groupBy(settings.freeExperiences, 'category')).map(([cat, items]) => (
                            <div key={cat} className="mb-4">
                                <h4 className="text-xs font-bold text-[#8C7B6C] mb-2 pl-1 border-l-2 border-[#C7B299]">{cat}</h4>
                                {items.map(s => (
                                    <div key={s.id} className="grid grid-cols-7 gap-2 items-center bg-[#F9F7F2] p-2 rounded-lg mb-2 text-sm">
                                        {editingId === s.id ? (
                                            <>
                                                <input className="col-span-2 p-1 border rounded" value={editForm.name} onChange={e=>setEditForm({...editForm, name: e.target.value})} />
                                                <div className="col-span-1 text-center text-gray-400 text-xs">-</div> {/* 免費無價格 */}
                                                <input className="col-span-1 p-1 border rounded" value={editForm.duration} onChange={e=>setEditForm({...editForm, duration: e.target.value})} />
                                                <div className="col-span-2"><CategorySelector items={settings.freeExperiences} current={editForm.category} onChange={val=>setEditForm({...editForm, category: val})} /></div>
                                                <div className="col-span-1 flex gap-1 justify-end">
                                                    <button onClick={() => saveEdit('free')} className="text-green-600"><Check size={16}/></button>
                                                    <button onClick={cancelEdit} className="text-red-400"><X size={16}/></button>
                                                </div>
                                            </>
                                        ) : (
                                            <>
                                                <div className="col-span-3 text-[#5E5049] truncate">{s.name}</div>
                                                <div className="col-span-1 text-xs text-gray-400">Free</div>
                                                <div className="col-span-1">{s.duration}分</div>
                                                <div className="col-span-2 text-right flex justify-end gap-2">
                                                    <button onClick={() => startEdit(s)} className="text-[#C7B299]"><Edit size={14}/></button>
                                                    <button onClick={() => removeFreeExp(s.id)} className="text-red-300"><Trash2 size={14}/></button>
                                                </div>
                                            </>
                                        )}
                                    </div>
                                ))}
                            </div>
                        ))}
                        <div className="grid grid-cols-7 gap-2 mt-3 pt-3 border-t border-[#E5D9CE]">
                             <div className="col-span-7 text-xs font-bold text-[#8C7B6C] mb-1">新增免費體驗</div>
                             <input className="col-span-2 p-2 border rounded-lg text-sm" placeholder="名稱" value={newFreeExp.name} onChange={e => setNewFreeExp({...newFreeExp, name: e.target.value})} />
                             <div className="col-span-1 flex items-center justify-center text-xs text-gray-400">Free</div>
                             <input className="col-span-1 p-2 border rounded-lg text-sm" placeholder="分" value={newFreeExp.duration} onChange={e => setNewFreeExp({...newFreeExp, duration: e.target.value})} />
                             <div className="col-span-2">
                                 <CategorySelector items={settings.freeExperiences} current={newFreeExp.category} onChange={val => setNewFreeExp({...newFreeExp, category: val})} />
                             </div>
                             <button className="col-span-1 bg-[#C7B299] text-white rounded-lg flex items-center justify-center" onClick={addFreeExp}><Plus size={18}/></button>
                        </div>
                    </Card>
                </>
            )}

            {subTab === 'times' && (
                <Card className="p-5 space-y-4">
                    <h3 className="font-bold text-[#5E5049] flex gap-2"><Clock size={18} /> 營業時段設定</h3>
                    <div className="flex flex-wrap gap-2">{settings.timeSlots.map(t => (<span key={t} className="bg-[#F9F7F2] text-[#5E5049] px-3 py-1 rounded-lg text-xs flex items-center gap-1">{t} <button onClick={() => updateSettings({ timeSlots: settings.timeSlots.filter(x => x !== t) })}><X size={12}/></button></span>))}</div>
                    <div className="flex gap-2"><input type="time" className="flex-1 p-2 border rounded-lg" value={newTimeSlot} onChange={e => setNewTimeSlot(e.target.value)} /><Button className="w-20" onClick={() => { if(newTimeSlot) updateSettings({ timeSlots: [...settings.timeSlots, newTimeSlot].sort() }); setNewTimeSlot(''); }}>新增</Button></div>
                </Card>
            )}
        </div>
    );
}

function StaffListView({ staff, addStaff, deleteStaff, onEdit, isAdmin, settings }) {
    const [isAddOpen, setIsAddOpen] = useState(false);
    const [newStaff, setNewStaff] = useState({ name: '', role: '', username: '', password: '123', commissionRate: 50 });
    const handleAdd = () => { if (!newStaff.name || !newStaff.username) return alert('請填寫完整'); addStaff({ ...newStaff }); setIsAddOpen(false); setNewStaff({ name: '', role: '', username: '', password: '123', commissionRate: 50 }); };
    
    // 使用 settings 中的 jobTitles (僅顯示名稱供選擇)
    const jobTitles = settings.jobTitles.map(t => t.name) || ['店長', '美甲師']; 

    return (
        <div className="space-y-4 animate-fadeIn">
            <Card className="p-5"><h3 className="font-bold text-[#5E5049] mb-4">新增人員</h3>{isAddOpen ? (<div className="space-y-3 animate-fadeIn"><div className="grid grid-cols-2 gap-2"><input placeholder="姓名" className="p-3 border rounded-xl" value={newStaff.name} onChange={e => setNewStaff({...newStaff, name: e.target.value, username: e.target.value})} /><select className="p-3 border rounded-xl bg-white" value={newStaff.role} onChange={e => setNewStaff({...newStaff, role: e.target.value})}><option value="">請選擇職稱</option>{jobTitles.map(r => <option key={r} value={r}>{r}</option>)}</select></div><div className="flex gap-2"><input type="number" placeholder="抽成 %" className="w-1/3 p-3 border rounded-xl" value={newStaff.commissionRate} onChange={e=>setNewStaff({...newStaff, commissionRate: e.target.value})} /><p className="text-xs self-center text-[#8C7B6C]">%</p></div><div className="flex gap-2 pt-2"><Button variant="outline" className="flex-1" onClick={() => setIsAddOpen(false)}>取消</Button><Button className="flex-1" onClick={handleAdd}>確認</Button></div></div>) : (<Button className="w-full" onClick={() => setIsAddOpen(true)}><UserPlus size={18}/> 新增人員</Button>)}</Card>
            <div className="space-y-3">{staff.map(s => (<Card key={s.id} className="p-4 flex justify-between items-center"><div className="flex items-center gap-3"><div className="w-10 h-10 rounded-full bg-[#EBE5DF] flex items-center justify-center text-[#8C7B6C] font-bold">{s.name[0]}</div><div><div className="font-bold text-[#5E5049]">{s.name}</div><div className="text-xs text-[#8C7B6C]">{s.role} {isAdmin && `| 抽成: ${s.commissionRate || 0}%`}</div></div></div><div className="flex gap-2"><button onClick={()=>onEdit(s)} className="p-2 text-[#C7B299] rounded-lg hover:bg-[#F9F7F2]"><Edit size={18}/></button><button onClick={() => { if(confirm('刪除?')) deleteStaff(s.id); }} className="p-2 text-red-300 rounded-lg hover:bg-[#F9F7F2]"><Trash2 size={18}/></button></div></Card>))}</div>
        </div>
    );
}

function AdminCalendarView({ appointments, onEdit }) {
    const [currentDate, setCurrentDate] = useState(new Date());
    const [selectedDate, setSelectedDate] = useState(new Date().toISOString().split('T')[0]);

    const year = currentDate.getFullYear();
    const month = currentDate.getMonth();
    const daysInMonth = new Date(year, month + 1, 0).getDate();
    const firstDay = new Date(year, month, 1).getDay();

    const prevMonth = () => setCurrentDate(new Date(year, month - 1, 1));
    const nextMonth = () => setCurrentDate(new Date(year, month + 1, 1));

    const getDayApps = (dateStr) => {
        return appointments.filter(app => app.date === dateStr && app.status !== 'cancelled')
                           .sort((a, b) => a.time.localeCompare(b.time));
    };

    return (
        <div className="space-y-6 animate-fadeIn">
            <Card className="p-5"><div className="flex justify-between items-center mb-4"><button onClick={prevMonth} className="p-2 hover:bg-[#F5F0EB] rounded-full"><ChevronLeft size={20} className="text-[#8C7B6C]"/></button><h3 className="font-bold text-lg text-[#5E5049]">{year} 年 {month + 1} 月</h3><button onClick={nextMonth} className="p-2 hover:bg-[#F5F0EB] rounded-full"><ChevronRight size={20} className="text-[#8C7B6C]"/></button></div><div className="grid grid-cols-7 gap-2 text-center mb-2 text-xs text-[#8C7B6C]">{['日','一','二','三','四','五','六'].map(d=><div key={d}>{d}</div>)}</div><div className="grid grid-cols-7 gap-2">{Array(firstDay).fill(null).map((_,i)=><div key={`e${i}`}/>)}{Array(daysInMonth).fill(null).map((_, i) => { const d = i + 1; const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`; const dayApps = getDayApps(dateStr); const isSelected = selectedDate === dateStr; const hasApps = dayApps.length > 0; return (<div key={d} onClick={() => setSelectedDate(dateStr)} className={`aspect-square rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all relative ${isSelected ? 'bg-[#C7B299] text-white shadow-md' : 'bg-white border border-[#E5D9CE] text-[#5E5049] hover:bg-[#F9F7F2]'}`}><span className="text-sm font-medium">{d}</span>{hasApps && (<div className={`w-1.5 h-1.5 rounded-full mt-1 ${isSelected ? 'bg-white' : 'bg-[#D4C3B3]'}`} />)}</div>); })}</div></Card>
            <div className="space-y-3 animate-fadeIn"><h4 className="font-bold text-[#5E5049] text-lg border-l-4 border-[#C7B299] pl-3">{selectedDate} 預約 ({getDayApps(selectedDate).length})</h4>{getDayApps(selectedDate).length === 0 ? (<div className="text-center text-[#8C7B6C] py-8 bg-white rounded-2xl border border-dashed border-[#E5D9CE]">本日無預約</div>) : (getDayApps(selectedDate).map(app => (<Card key={app.id} className="p-4 flex justify-between items-center"><div><div className="flex items-center gap-2 mb-1"><span className="bg-[#F5F0EB] text-[#5E5049] font-bold px-2 py-0.5 rounded-md text-xs">{app.time}</span><span className="font-bold text-[#5E5049]">{app.name}</span></div><div className="text-xs text-[#8C7B6C]">{app.mainService?.name} {app.staff ? `(${app.staff.name})` : ''}</div></div><Button variant="outline" className="px-3 py-1 text-xs" onClick={() => onEdit(app)}><Edit size={14} /></Button></Card>)))}</div>
        </div>
    );
}

function EditAppointmentModal({ appointment, settings, staff, members, onClose, onSave }) {
    const [form, setForm] = useState({ ...appointment });
    const [searchTerm, setSearchTerm] = useState('');
    const [showResults, setShowResults] = useState(false);
    const searchResults = searchTerm ? members.filter(m => m.name.includes(searchTerm) || m.phone.includes(searchTerm)) : [];
    const selectMember = (m) => { setForm({ ...form, name: m.name, phone: m.phone, lineId: m.lineId }); setSearchTerm(''); setShowResults(false); };
    const calculateTotal = (main, adds) => { let total = main ? Number(main.price) : 0; if (adds) adds.forEach(a => total += Number(a.price)); return total; };
    const handleSave = () => { onSave(appointment.id, { ...form, totalPrice: calculateTotal(form.mainService, form.addOns) }); };
    
    return (
        <div className="fixed inset-0 bg-black/40 backdrop-blur-sm z-50 flex items-end sm:items-center justify-center p-0 sm:p-4">
            <div className="bg-white w-full max-w-lg rounded-t-3xl sm:rounded-3xl shadow-2xl p-6 space-y-4 max-h-[90vh] overflow-y-auto custom-scrollbar animate-fadeIn">
                <div className="flex justify-between items-center border-b border-[#E5D9CE] pb-3"><h3 className="font-bold text-lg text-[#5E5049]">{appointment.id ? '編輯預約' : '新增預約'}</h3><button onClick={onClose} className="text-[#8C7B6C]"><X /></button></div>
                
                <div className="grid grid-cols-2 gap-4"><div><label className="text-xs font-bold text-[#8C7B6C]">日期</label><input type="date" className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={form.date} onChange={e => setForm({...form, date: e.target.value})} /></div><div><label className="text-xs font-bold text-[#8C7B6C]">時間</label><select className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={form.time} onChange={e => setForm({...form, time: e.target.value})}>{settings.timeSlots.map(t => <option key={t} value={t}>{t}</option>)}</select></div></div>
                <div><label className="text-xs font-bold text-[#8C7B6C]">主課程</label><select className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={form.mainService?.id} onChange={e => { const s = settings.mainServices.find(x => x.id === e.target.value); setForm({...form, mainService: s}); }}>{settings.mainServices.map(s => <option key={s.id} value={s.id}>{s.name} (${s.price})</option>)}</select></div>
                <div><label className="text-xs font-bold text-[#8C7B6C]">服務人員</label><select className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={form.staff?.id || ''} onChange={e => { const s = staff.find(x => x.id === e.target.value); setForm({...form, staff: s || null}); }}><option value="">不指定</option>{staff.map(s => <option key={s.id} value={s.id}>{s.name}</option>)}</select></div>
                <div><label className="text-xs font-bold text-[#8C7B6C] mb-1 block">加購</label><div className="flex flex-wrap gap-2">{settings.paidAddons.map(addon => (<button key={addon.id} onClick={() => { const exists = form.addOns?.find(a => a.id === addon.id); const newAdd = exists ? form.addOns.filter(a => a.id !== addon.id) : [...(form.addOns || []), addon]; setForm({...form, addOns: newAdd}); }} className={`px-3 py-1 rounded-lg text-xs border ${form.addOns?.find(a => a.id === addon.id) ? 'bg-[#C7B299] text-white border-[#C7B299]' : 'bg-white text-[#8C7B6C] border-[#E5D9CE]'}`}>{addon.name}</button>))}</div></div>
                
                {/* 新增：姓名電話欄位 (支援快搜) */}
                {!appointment.id && (
                    <div className="relative space-y-2 p-3 bg-[#FDFBF7] rounded-xl border border-[#E5D9CE]">
                        <div className="flex justify-between items-center mb-1"><label className="text-xs font-bold text-[#8C7B6C]">快速搜尋會員</label></div>
                        <div className="relative">
                             <Search className="absolute left-3 top-2.5 text-[#8C7B6C] w-4 h-4" />
                             <input 
                                type="text" 
                                className="w-full pl-9 p-2 border border-[#E5D9CE] rounded-lg bg-white text-sm" 
                                placeholder="輸入姓名或電話搜尋..." 
                                value={searchTerm}
                                onChange={(e) => { setSearchTerm(e.target.value); setShowResults(true); }}
                             />
                             {showResults && searchTerm && (
                                 <div className="absolute z-10 w-full bg-white border border-[#E5D9CE] rounded-lg shadow-lg mt-1 max-h-40 overflow-y-auto">
                                     {searchResults.length > 0 ? (
                                         searchResults.map(m => (
                                             <div key={m.id} onClick={() => selectMember(m)} className="p-2 hover:bg-[#F9F7F2] cursor-pointer text-sm border-b border-[#F9F7F2] last:border-0">
                                                 <div className="font-bold text-[#5E5049]">{m.name}</div>
                                                 <div className="text-xs text-[#8C7B6C]">{m.phone}</div>
                                             </div>
                                         ))
                                     ) : (
                                         <div className="p-2 text-xs text-[#8C7B6C] text-center">查無會員</div>
                                     )}
                                 </div>
                             )}
                        </div>
                        <div className="grid grid-cols-2 gap-4 pt-2">
                             <div><label className="text-xs font-bold text-[#8C7B6C]">姓名</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.name} onChange={e => setForm({...form, name: e.target.value})} /></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">電話</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.phone} onChange={e => setForm({...form, phone: e.target.value})} /></div>
                        </div>
                    </div>
                )}

                <div><label className="text-xs font-bold text-[#8C7B6C]">備註</label><textarea className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2] h-20" value={form.notes || ''} onChange={e => setForm({...form, notes: e.target.value})} /></div>
                <div><label className="text-xs font-bold text-[#8C7B6C]">狀態</label><select className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-[#F9F7F2]" value={form.status} onChange={e => setForm({...form, status: e.target.value})}><option value="pending">待處理</option><option value="completed">已完成</option><option value="cancelled">已取消</option></select></div>
                <div className="pt-2 border-t border-[#E5D9CE] flex justify-between items-center"><span className="text-[#8C7B6C]">總金額: <span className="font-bold text-lg text-[#C7B299]">${calculateTotal(form.mainService, form.addOns)}</span></span><Button onClick={handleSave}>儲存</Button></div>
            </div>
        </div>
    );
}

function EditMemberModal({ member, onClose, onSave, staff }) {
    const [form, setForm] = useState({ ...member });
    return (
        <div className="fixed inset-0 bg-black/40 backdrop-blur-sm z-50 flex items-end sm:items-center justify-center p-0 sm:p-4">
            <div className="bg-white w-full max-w-lg rounded-t-3xl sm:rounded-3xl shadow-2xl p-6 space-y-4 max-h-[90vh] overflow-y-auto custom-scrollbar animate-fadeIn">
                <div className="flex justify-between items-center border-b border-[#E5D9CE] pb-3"><h3 className="font-bold text-lg text-[#5E5049]">{member.id ? '編輯會員' : '新增會員'}</h3><button onClick={onClose}><X /></button></div>
                <div className="grid grid-cols-2 gap-4">
                    <div><label className="text-xs font-bold text-[#8C7B6C]">姓名</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.name} onChange={e => setForm({...form, name: e.target.value})} /></div>
                    <div><label className="text-xs font-bold text-[#8C7B6C]">電話</label><input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.phone} onChange={e => setForm({...form, phone: e.target.value})} /></div>
                </div>
                <div className="space-y-1">
                    <label className="text-xs font-bold text-[#8C7B6C]">LINE ID / 名稱</label>
                    <input className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.lineId || ''} onChange={e => setForm({...form, lineId: e.target.value})} placeholder="請輸入 LINE ID" />
                </div>
                {/* 新增：慣用操作員 */}
                <div className="space-y-1">
                    <label className="text-xs font-bold text-[#8C7B6C]">慣用操作員</label>
                    <select 
                        className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" 
                        value={form.preferredStaffId || ''} 
                        onChange={e => setForm({...form, preferredStaffId: e.target.value})}
                    >
                        <option value="">無特別指定</option>
                        {staff.map(s => (
                            <option key={s.id} value={s.id}>{s.name} ({s.role})</option>
                        ))}
                    </select>
                </div>
                <div className="grid grid-cols-2 gap-4">
                    <div><label className="text-xs font-bold text-[#8C7B6C]">等級</label><select className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.level || 'Bronze'} onChange={e => setForm({...form, level: e.target.value})}><option value="Bronze">一般</option><option value="Silver">銀卡</option><option value="Gold">金卡</option></select></div>
                    <div><label className="text-xs font-bold text-[#8C7B6C]">點數</label><input type="number" className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white" value={form.points || 0} onChange={e => setForm({...form, points: parseInt(e.target.value)||0})} /></div>
                </div>
                <div><label className="text-xs font-bold text-[#8C7B6C]">備註</label><textarea className="w-full p-2 border border-[#E5D9CE] rounded-lg bg-white h-20" value={form.adminNotes || ''} onChange={e => setForm({...form, adminNotes: e.target.value})} /></div>
                <div className="pt-2 border-t border-[#E5D9CE]"><Button onClick={() => onSave(member.id, form)}>儲存</Button></div>
            </div>
        </div>
    );
}

function MemberHistoryModal({ member, appointments, onClose, onAddRecord, settings, staff }) {
    const [isAdding, setIsAdding] = useState(false);
    const [manualForm, setManualForm] = useState({
        date: new Date().toISOString().split('T')[0],
        name: member.name,
        phone: member.phone,
        mainServiceId: '',
        price: '',
        staffId: ''
    });

    const history = appointments.filter(app => app.phone.replace(/\D/g, '') === member.phone.replace(/\D/g, '')).sort((a, b) => new Date(b.date + 'T' + b.time) - new Date(a.date + 'T' + a.time));
    
    const handleAdd = () => {
        if(!manualForm.price || !manualForm.mainServiceId) return alert('請填寫完整資訊');
        const service = settings.mainServices.find(s => s.id === manualForm.mainServiceId);
        const selectedStaff = staff.find(s => s.id === manualForm.staffId);
        
        onAddRecord({
            ...manualForm,
            mainService: service || { name: '自訂項目', price: manualForm.price },
            totalPrice: Number(manualForm.price),
            staff: selectedStaff || null
        });
        setIsAdding(false);
    };

    return (
        <div className="fixed inset-0 bg-black/40 backdrop-blur-sm z-50 flex items-center justify-center p-4">
            <div className="bg-white w-full max-w-md rounded-2xl shadow-2xl p-0 overflow-hidden flex flex-col max-h-[80vh] animate-fadeIn">
                <div className="p-4 border-b border-[#E5D9CE] flex justify-between items-center bg-[#F9F7F2]"><div><h3 className="font-bold text-lg text-[#5E5049]">{member.name} 的消費紀錄</h3><p className="text-xs text-[#8C7B6C]">{member.phone}</p></div><button onClick={onClose} className="p-2 hover:bg-[#EBE5DF] rounded-full"><X size={20} className="text-[#8C7B6C]"/></button></div>
                
                <div className="flex-1 overflow-y-auto p-4 space-y-3 custom-scrollbar">
                    {isAdding ? (
                        <div className="bg-[#F9F7F2] p-4 rounded-xl space-y-3 animate-fadeIn border border-[#E5D9CE]">
                             <div className="flex justify-between items-center mb-2"><h4 className="font-bold text-[#5E5049]">新增紀錄</h4><button onClick={() => setIsAdding(false)}><X size={16}/></button></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">日期</label><input type="date" className="w-full p-2 border rounded-lg bg-white" value={manualForm.date} onChange={e=>setManualForm({...manualForm, date: e.target.value})} /></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">項目</label><select className="w-full p-2 border rounded-lg bg-white" value={manualForm.mainServiceId} onChange={e=>{
                                 const s = settings.mainServices.find(x => x.id === e.target.value);
                                 setManualForm({...manualForm, mainServiceId: e.target.value, price: s ? s.price : manualForm.price});
                             }}><option value="">請選擇項目</option>{settings.mainServices.map(s => <option key={s.id} value={s.id}>{s.name}</option>)}</select></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">金額</label><input type="number" className="w-full p-2 border rounded-lg bg-white" value={manualForm.price} onChange={e=>setManualForm({...manualForm, price: e.target.value})} /></div>
                             <div><label className="text-xs font-bold text-[#8C7B6C]">人員</label><select className="w-full p-2 border rounded-lg bg-white" value={manualForm.staffId} onChange={e=>setManualForm({...manualForm, staffId: e.target.value})}><option value="">不指定</option>{staff.map(s => <option key={s.id} value={s.id}>{s.name}</option>)}</select></div>
                             <Button onClick={handleAdd} className="w-full mt-2">確認新增</Button>
                        </div>
                    ) : (
                        <Button variant="outline" className="w-full border-dashed mb-2" onClick={() => setIsAdding(true)}><Plus size={16}/> 新增紀錄</Button>
                    )}

                    {history.length === 0 ? (<div className="text-center py-8 text-[#8C7B6C]">尚無消費紀錄</div>) : (history.map(app => (<div key={app.id} className="bg-[#F9F7F2] p-3 rounded-xl border border-[#E5D9CE]"><div className="flex justify-between mb-1"><span className="font-bold text-[#5E5049] text-sm">{app.date} {app.time !== '00:00' ? app.time : ''}</span><span className="font-bold text-[#C7B299]">${app.totalPrice}</span></div><div className="text-sm text-[#5E5049] mb-1">{app.mainService?.name}</div><div className="flex justify-between items-center text-xs text-[#8C7B6C] border-t border-[#E5D9CE] pt-2 mt-1"><span>狀態: {app.status === 'completed' ? '完成' : app.status === 'cancelled' ? '取消' : '待處理'}</span><span className="bg-white px-2 py-0.5 rounded border border-[#E5D9CE] flex items-center gap-1"><User size={10} /> {app.staff ? app.staff.name : '未指定'}</span></div></div>)))}
                </div>
            </div>
        </div>
    );
}

function RosterView({ staff, roster, updateRoster, settings }) {
    const [viewMode, setViewMode] = useState('staff'); // 'staff' | 'day'
    const [targetStaff, setTargetStaff] = useState(staff[0]);
    const [currentDate, setCurrentDate] = useState(new Date());
    const [selectedDate, setSelectedDate] = useState(null); 
    const year = currentDate.getFullYear();
    const month = currentDate.getMonth();
    const daysInMonth = new Date(year, month + 1, 0).getDate();
    const firstDay = new Date(year, month, 1).getDay();
    const getDateSettings = (d) => {
        const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`;
        const entry = roster.find(r => r.date === dateStr && r.staffId === targetStaff?.id);
        return { dateStr, entry };
    };
    const getStaffRoster = (dateStr, staffId) => {
        const entry = roster.find(r => r.date === dateStr && r.staffId === staffId);
        return entry || { status: 'work', slots: settings.timeSlots };
    };
    const getDailyWorkingCount = (day) => {
        const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
        return staff.filter(s => {
            const { status } = getStaffRoster(dateStr, s.id);
            return status === 'work';
        }).length;
    };
    const handleUpdate = (type, val) => {
        if (!selectedDate || !targetStaff) return;
        const { dateStr, entry } = getDateSettings(selectedDate);
        let newData = { status: entry?.status || 'work', slots: entry?.slots || settings.timeSlots };
        if (type === 'status') { newData.status = val; if (val === 'off') newData.slots = []; else newData.slots = settings.timeSlots; } 
        else if (type === 'slot') { if (newData.slots.includes(val)) newData.slots = newData.slots.filter(s => s !== val); else newData.slots = [...newData.slots, val].sort(); if (newData.slots.length > 0) newData.status = 'work'; }
        updateRoster(dateStr, targetStaff.id, newData);
    };
    const handleRosterUpdate = (dateStr, staffId, newStatus, newSlots) => {
        updateRoster(dateStr, staffId, { status: newStatus, slots: newSlots });
    };

    return (
        <div className="space-y-6 animate-fadeIn">
             <div className="flex p-1 bg-white border border-[#E5D9CE] rounded-xl shadow-sm"><button onClick={()=>{setViewMode('staff'); setSelectedDate(null);}} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all ${viewMode==='staff' ? 'bg-[#C7B299] text-white shadow' : 'text-[#8C7B6C]'}`}>依人員排班</button><button onClick={()=>{setViewMode('day'); setSelectedDate(null);}} className={`flex-1 py-2 text-xs font-bold rounded-lg transition-all ${viewMode==='day' ? 'bg-[#C7B299] text-white shadow' : 'text-[#8C7B6C]'}`}>每日總覽</button></div>
            {viewMode === 'staff' && (<><div className="bg-white p-4 rounded-2xl border border-[#E5D9CE] shadow-sm"><h3 className="text-sm font-bold text-[#8C7B6C] mb-3">設定對象</h3><div className="flex gap-2 overflow-x-auto no-scrollbar">{staff.map(s => (<button key={s.id} onClick={() => { setTargetStaff(s); setSelectedDate(null); }} className={`flex items-center gap-2 px-4 py-2 rounded-xl border whitespace-nowrap ${targetStaff?.id === s.id ? 'bg-[#C7B299] text-white' : 'text-[#8C7B6C]'}`}><User size={16} /> {s.name}</button>))}</div></div>{targetStaff && (<div className="bg-white p-5 rounded-2xl border border-[#E5D9CE] shadow-sm"><div className="flex justify-between items-center mb-4"><span className="text-lg font-bold text-[#5E5049]">{year} 年 {month + 1} 月</span><div className="flex gap-3 text-xs"><span className="flex items-center gap-1"><div className="w-2 h-2 rounded-full bg-green-400"/>上班</span><span className="flex items-center gap-1"><div className="w-2 h-2 rounded-full bg-stone-300"/>休假</span></div></div><div className="grid grid-cols-7 gap-2 text-center mb-2 text-xs text-[#8C7B6C]">{['日','一','二','三','四','五','六'].map(d=><div key={d}>{d}</div>)}</div><div className="grid grid-cols-7 gap-2">{Array(firstDay).fill(null).map((_,i)=><div key={`e${i}`}/>)}{Array(daysInMonth).fill(null).map((_, i) => { const d = i + 1; const { entry } = getDateSettings(d); const status = entry?.status || 'work'; const isSelected = selectedDate === d; return (<div key={d} onClick={() => setSelectedDate(d)} className={`aspect-square rounded-xl flex flex-col items-center justify-center cursor-pointer border-2 transition-all ${isSelected ? 'border-[#d86c9c] bg-pink-50' : 'border-transparent hover:bg-[#F9F7F2]'} ${status === 'work' ? '' : 'opacity-50 grayscale'}`}><span className="text-sm font-medium text-[#5E5049]">{d}</span><div className={`w-1.5 h-1.5 rounded-full mt-1 ${status === 'work' ? 'bg-green-400' : 'bg-stone-300'}`} /></div>); })}</div>{selectedDate && <div className="mt-6 pt-6 border-t border-[#E5D9CE]"><div className="space-y-4 animate-fadeIn"><div className="flex justify-between items-center"><h4 className="font-bold text-[#5E5049]">{month+1}月{selectedDate}日 排班 ({targetStaff.name})</h4><div className="flex bg-[#F5F0EB] p-1 rounded-lg"><button onClick={()=>handleUpdate('status', 'off')} className={`px-3 py-1 text-xs rounded-md transition-all ${getDateSettings(selectedDate).entry?.status==='off' ? 'bg-stone-600 text-white shadow-md' : 'text-[#8C7B6C] hover:bg-white'}`}>全天休</button><button onClick={()=>handleUpdate('status', 'work')} className={`px-3 py-1 text-xs rounded-md transition-all ${getDateSettings(selectedDate).entry?.status==='work' || !getDateSettings(selectedDate).entry ? 'bg-green-100 text-green-700 shadow-md' : 'text-[#8C7B6C] hover:bg-white'}`}>全天班</button></div></div><div className="grid grid-cols-4 gap-2">{settings.timeSlots.map(t => (<button key={t} onClick={() => handleUpdate('slot', t)} className={`py-2 text-xs rounded-lg border ${getDateSettings(selectedDate).entry?.slots?.includes(t) ? 'bg-green-50 border-green-300' : ''}`}>{t}</button>))}</div></div></div>}</div>)}</>)}
            {viewMode === 'day' && (<div className="bg-white p-5 rounded-2xl border border-[#E5D9CE] shadow-sm"><div className="flex justify-between items-center mb-4"><h3 className="text-lg font-bold text-[#5E5049]">{year} 年 {month + 1} 月</h3><div className="text-xs text-[#8C7B6C] 點擊日期查看詳情"></div></div><div className="grid grid-cols-7 gap-2 text-center mb-2 text-xs text-[#8C7B6C]">{['日','一','二','三','四','五','六'].map(d=><div key={d}>{d}</div>)}</div><div className="grid grid-cols-7 gap-2">{Array(firstDay).fill(null).map((_,i)=><div key={`e${i}`}/>)}{Array(daysInMonth).fill(null).map((_, i) => { const d = i + 1; const count = getDailyWorkingCount(d); const isSelected = selectedDate === d; return (<div key={d} onClick={() => setSelectedDate(d)} className={`aspect-square rounded-xl flex flex-col items-center justify-center cursor-pointer border-2 transition-all ${isSelected ? 'border-[#C7B299] bg-[#F9F7F2]' : 'border-transparent hover:bg-[#F9F7F2]'}`}><span className="text-sm font-medium text-[#5E5049]">{d}</span><span className="text-[10px] bg-[#C7B299] text-white px-1.5 rounded-full mt-1">👤 {count}</span></div>); })}</div>{selectedDate && (<div className="mt-6 pt-6 border-t border-[#E5D9CE] animate-fadeIn"><div className="flex justify-between items-center mb-4"><h4 className="font-bold text-[#5E5049]">{month+1}月{selectedDate}日 人員排班 ({getDailyWorkingCount(selectedDate)}人上班)</h4></div><div className="space-y-3">{staff.map(s => { const dateStr = `${year}-${String(month+1).padStart(2,'0')}-${String(selectedDate).padStart(2,'0')}`; const { status, slots } = getStaffRoster(dateStr, s.id); return (<div key={s.id} className="flex flex-col bg-[#F9F7F2] p-3 rounded-xl border border-[#E5D9CE]"><div className="flex justify-between items-center mb-2"><div className="font-bold text-[#5E5049] flex items-center gap-2"><User size={16}/> {s.name}</div><div className="flex bg-white p-0.5 rounded-lg border border-[#E5D9CE]"><button onClick={()=>handleRosterUpdate(dateStr, s.id, 'off', [])} className={`px-3 py-1 text-xs rounded-md transition-all ${status==='off'?'bg-stone-600 text-white shadow-md':'text-[#8C7B6C] hover:bg-[#F5F0EB]'}`}>休</button><button onClick={()=>handleRosterUpdate(dateStr, s.id, 'work', settings.timeSlots)} className={`px-3 py-1 text-xs rounded-md transition-all ${status==='work'?'bg-green-100 text-green-700 shadow-md':'text-[#8C7B6C] hover:bg-[#F5F0EB]'}`}>班</button></div></div>{status === 'work' && (<div className="flex flex-wrap gap-1">{settings.timeSlots.map(t => { const isWork = (slots || settings.timeSlots).includes(t); return (<button key={t} onClick={()=>{ const newSlots = isWork ? slots.filter(sl=>sl!==t) : [...(slots||settings.timeSlots), t].sort(); handleRosterUpdate(dateStr, s.id, 'work', newSlots); }} className={`text-[10px] px-2 py-1 rounded border ${isWork ? 'bg-green-50 border-green-200 text-green-700' : 'bg-white border-stone-200 text-stone-300'}`}>{t}</button>); })}</div>)}</div>); })}</div></div>)}</div>)}
        </div>
    );
}

function BookingListView({ appointments, updateAppointment, deleteAppointment, onEdit }) {
    return (
        <div className="space-y-4 animate-fadeIn">
            {appointments.map(app => (
                 <Card key={app.id} className="p-5">
                    <div className="flex justify-between items-start mb-2"><div><div className="font-bold text-[#5E5049] text-lg flex items-center gap-2">{app.name} <Tag status={app.status} /></div><div className="text-sm text-[#8C7B6C]">{app.phone}</div></div><div className="text-right font-bold text-[#C7B299]">${app.totalPrice}</div></div>
                    <div className="bg-[#F9F7F2] p-3 rounded-xl text-sm space-y-1 mt-3 text-[#5E5049]"><div className="font-bold flex items-center gap-2"><Calendar size={14}/> {app.date} {app.time !== '00:00' ? app.time : '手動紀錄'}</div><div>{app.mainService?.name}</div>{app.staff && <div className="text-xs text-[#8C7B6C]">人員: {app.staff.name}</div>}</div>
                    <div className="flex gap-2 mt-3"><Button variant="outline" className="flex-1 text-xs py-2" onClick={() => onEdit(app)}><Edit size={14} className="mr-1"/> 編輯</Button><button onClick={()=>{if(confirm('確定刪除?')) deleteAppointment(app.id)}} className="p-2 text-red-300 hover:bg-red-50 rounded-lg"><Trash2 size={16}/></button></div>
                 </Card>
            ))}
        </div>
    );
}

function MemberListView({ members, updatePoints, onEdit, onViewHistory, onAdd }) {
    const [search, setSearch] = useState('');
    const filtered = members.filter(m => m.name.includes(search) || m.phone.includes(search));
    return (
        <div className="space-y-4 animate-fadeIn">
            <div className="flex gap-2 items-center">
                <div className="relative flex-1">
                    <Search className="absolute left-4 top-3.5 text-[#8C7B6C] w-5 h-5" />
                    <input 
                        placeholder="搜尋會員..." 
                        className="w-full pl-12 p-3.5 h-[52px] rounded-2xl border border-[#E5D9CE] outline-none" 
                        value={search} 
                        onChange={e => setSearch(e.target.value)} 
                    />
                </div>
                <Button variant="primary" className="h-[52px] px-6 whitespace-nowrap" onClick={onAdd}><UserPlus size={18}/> 新增</Button>
            </div>
            <div className="space-y-3">{filtered.map(m => (<Card key={m.id} className="p-5"><div className="flex justify-between items-start mb-4"><div><div className="flex items-center gap-2 mb-1"><h3 className="text-lg font-bold text-[#5E5049]">{m.name}</h3><Tag status={m.level || 'Bronze'}>{m.level || 'Bronze'}</Tag></div><p className="text-[#8C7B6C]">{m.phone}</p>{m.lineId && <p className="text-xs text-[#C7B299] mt-1">LINE: {m.lineId}</p>}</div><div className="text-right"><div className="text-2xl font-bold text-[#8B6F56]">{m.points || 0}</div><div className="text-xs text-[#8C7B6C]">點數</div></div></div><div className="flex justify-between items-center pt-4 border-t border-[#EBE5DF]"><span className="text-xs text-[#8C7B6C]">來訪: {m.lastVisit ? m.lastVisit.split('T')[0] : 'New'}</span><div className="flex gap-2"><Button variant="outline" className="px-3 py-1 text-xs" onClick={() => onViewHistory(m)}><Eye size={14} /> 紀錄</Button><Button variant="outline" className="px-3 py-1 text-xs" onClick={() => onEdit(m)}><Edit size={14} /> 編輯</Button></div></div></Card>))}</div>
        </div>
    );
}

function BookingFlow({ onBook, settings, staff, roster, members }) {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({ mainService: null, freeServices: [], addOns: [], date: '', time: '', name: '', phone: '', lineId: '', notes: '', staff: null });
  const [agreedToRules, setAgreedToRules] = useState(false);
  const calculateTotal = () => { let total = formData.mainService ? Number(formData.mainService.price) : 0; formData.addOns.forEach(a => total += Number(a.price)); return total; };
  const handleSubmit = async () => { if (!formData.name || !formData.phone) return alert('請填寫完整資料'); if (!agreedToRules) return alert('請同意預約須知'); await onBook({...formData, totalPrice: calculateTotal()}); setStep(5); };
  
  const handlePhoneChange = (e) => {
      const inputPhone = e.target.value;
      setFormData({...formData, phone: inputPhone});
      
      if (inputPhone.length >= 4) {
          const cleanInput = inputPhone.replace(/\D/g, '');
          const foundMember = members.find(m => m.phone.replace(/\D/g, '') === cleanInput);
          if (foundMember) {
              setFormData(prev => ({
                  ...prev,
                  phone: inputPhone,
                  name: foundMember.name,
                  lineId: foundMember.lineId || prev.lineId
              }));
          }
      }
  };

  const isStaffAvailable = (s) => {
      if (!formData.date || !formData.time) return true; 
      const entry = roster.find(r => r.date === formData.date && r.staffId === s.id);
      if (entry) {
          if (entry.status === 'off') return false;
          if (entry.slots && !entry.slots.includes(formData.time)) return false;
      }
      return true; 
  };
  
  const handleReset = () => {
      setStep(1);
      setFormData({ mainService: null, freeServices: [], addOns: [], date: '', time: '', name: '', phone: '', lineId: '', notes: '', staff: null });
      setAgreedToRules(false); 
  };
  
  const handleCopyToLine = () => {
      // 1. Content construction
      const content = `你好，我想預約：
姓名：${formData.name}
電話：${formData.phone}
預約時間：${formData.date} ${formData.time}
預約項目：${formData.mainService?.name}${formData.addOns.length > 0 ? ` (+ ${formData.addOns.map(a => a.name).join(', ')})` : ''}
備註：${formData.notes || '無'}`;

      // 2. URL construction (The "magic" link)
      // 使用 oaMessage 可以在加入好友後直接帶入訊息
      const lineId = '@287gcjjf';
      const deepLink = `https://line.me/R/oaMessage/${lineId}/?${encodeURIComponent(content)}`;

      // 3. Execution wrapper
      const performAction = () => {
           // Attempt to open
           window.open(deepLink, '_blank');
      };

      // 4. Copy to clipboard as backup then open
      if (navigator.clipboard && navigator.clipboard.writeText) {
          navigator.clipboard.writeText(content)
              .then(() => {
                  alert('預約資料已複製！\n即將開啟 LINE，請選擇您的官方帳號。\n(若訊息未自動帶入，請在對話框長按貼上)');
                  performAction();
              })
              .catch(() => {
                  performAction(); // Fail silently on copy, just open
              });
      } else {
          // Fallback copy
          try {
            const textArea = document.createElement("textarea");
            textArea.value = content;
            document.body.appendChild(textArea);
            textArea.select();
            document.execCommand('copy');
            document.body.removeChild(textArea);
            alert('預約資料已複製！\n即將開啟 LINE，請選擇您的官方帳號。\n(若訊息未自動帶入，請在對話框長按貼上)');
          } catch (e) {}
          performAction();
      }
  };

  if (step === 5) return (
    <div className="flex flex-col items-center justify-center min-h-[60vh] space-y-6 animate-fadeIn p-4">
        <div className="w-24 h-24 bg-[#C7B299] text-white rounded-full flex items-center justify-center shadow-lg">
            <Check size={48} />
        </div>
        <div className="text-center space-y-2">
            <h2 className="text-2xl font-bold text-[#5E5049]">預約已送出！</h2>
            <p className="text-[#8C7B6C] text-sm">請點擊下方按鈕，透過 LINE 確認預約</p>
        </div>
        
        <Card className="w-full max-w-sm p-4 bg-[#F9F7F2] border border-[#E5D9CE] text-sm text-[#5E5049] space-y-2">
             <div className="font-bold border-b border-[#E5D9CE] pb-2 mb-2 text-center">預約明細</div>
             <p>姓名：{formData.name}</p>
             <p>電話：{formData.phone}</p>
             <p>時間：{formData.date} {formData.time}</p>
             <p>項目：{formData.mainService?.name} {formData.addOns.length > 0 && `(+${formData.addOns.length})`}</p>
        </Card>

        <Button className="w-full max-w-xs bg-[#06C755] hover:bg-[#05b34c] text-white border-none shadow-lg animate-bounce" onClick={handleCopyToLine}>
            <MessageCircle size={20} className="mr-2"/> 複製預約並前往 LINE
        </Button>

        {/* 備用連結：如果自動跳轉失敗 */}
        <a href={`https://line.me/R/oaMessage/@287gcjjf/?${encodeURIComponent(`你好，我想預約：
姓名：${formData.name}
電話：${formData.phone}
預約時間：${formData.date} ${formData.time}
預約項目：${formData.mainService?.name}${formData.addOns.length > 0 ? ` (+ ${formData.addOns.map(a => a.name).join(', ')})` : ''}
備註：${formData.notes || '無'}`)}`} target="_blank" rel="noopener noreferrer" className="text-xs text-[#06C755] underline mt-2 block">
            如果沒有自動開啟，請點此前往 LINE
        </a>
        
        <Button variant="text" className="text-sm mt-4" onClick={handleReset}>返回首頁</Button>
    </div>
  );

  return (
    <div className="space-y-6 animate-fadeIn p-4 pb-24">
        <div className="flex justify-between mb-4 px-2">{[1, 2, 3, 4].map(i => (<div key={i} className={`h-1.5 flex-1 mx-1 rounded-full transition-all ${step >= i ? 'bg-[#C7B299]' : 'bg-[#E5D9CE]'}`} />))}</div>
        
        {step === 1 && (
            <div className="space-y-4 animate-fadeIn">
                <h2 className="text-xl font-bold text-[#5E5049]">選擇服務</h2>
                {Object.entries(groupBy(settings.mainServices, 'category')).map(([cat, items]) => (
                    <div key={cat}>
                        <h4 className="text-sm font-bold text-[#8C7B6C] mb-2 border-l-4 border-[#C7B299] pl-2">{cat}</h4>
                        <div className="space-y-3">
                            {items.map(s => (
                                <Card key={s.id} onClick={() => setFormData({...formData, mainService: s})} className={`p-5 cursor-pointer transition-all ${formData.mainService?.id === s.id ? 'border-[#C7B299] ring-1 ring-[#C7B299]' : 'hover:border-[#C7B299]'}`}>
                                    <div className="flex justify-between">
                                        <div><div className="font-bold text-[#5E5049]">{s.name}</div><div className="text-sm text-[#8C7B6C]">{s.duration} 分</div></div>
                                        <div className="text-[#C7B299] font-bold">${s.price}</div>
                                    </div>
                                </Card>
                            ))}
                        </div>
                    </div>
                ))}
                <Button className="w-full mt-4" onClick={() => setStep(2)} disabled={!formData.mainService}>下一步</Button>
            </div>
        )}

        {step === 2 && (
            <div className="space-y-6 animate-fadeIn">
                <button onClick={() => setStep(1)} className="flex items-center text-[#8C7B6C] mb-2"><ChevronLeft size={16}/> 返回上一步</button>
                
                <div>
                    <h3 className="text-lg font-bold text-[#5E5049] mb-3">加購</h3>
                    {Object.entries(groupBy(settings.paidAddons, 'category')).map(([cat, items]) => (
                        <div key={cat} className="mb-4">
                            <h4 className="text-xs font-bold text-[#8C7B6C] mb-2">{cat}</h4>
                            <div className="space-y-3">
                                {items.map(addon => (
                                    <Card key={addon.id} onClick={() => { const exists = formData.addOns.find(a=>a.id===addon.id); setFormData({...formData, addOns: exists ? formData.addOns.filter(a=>a.id!==addon.id) : [...formData.addOns, addon]}); }} className={`p-4 cursor-pointer flex justify-between ${formData.addOns.find(a=>a.id===addon.id) ? 'border-[#C7B299]' : ''}`}>
                                            <div className="text-[#5E5049]">{addon.name} <span className="text-xs">(${addon.price})</span></div>
                                            {formData.addOns.find(a=>a.id===addon.id) && <Check size={18} className="text-[#C7B299]"/>}
                                    </Card>
                                ))}
                            </div>
                        </div>
                    ))}
                </div>

                <div>
                    <h3 className="text-lg font-bold text-[#5E5049] mb-3">免費體驗 (可複選)</h3>
                    {Object.entries(groupBy(settings.freeExperiences, 'category')).map(([cat, items]) => (
                        <div key={cat} className="mb-4">
                            <h4 className="text-xs font-bold text-[#8C7B6C] mb-2">{cat}</h4>
                            <div className="grid grid-cols-2 gap-3">
                                {items.map(item => (
                                    <div key={item.id} onClick={() => { const exists = formData.freeServices.includes(item.name); setFormData({...formData, freeServices: exists ? formData.freeServices.filter(i=>i!==item.name) : [...formData.freeServices, item.name]}); }} className={`p-3 rounded-xl border text-sm text-center cursor-pointer flex flex-col justify-center items-center ${formData.freeServices.includes(item.name) ? 'border-[#C7B299] bg-[#FAF9F6]' : 'border-[#E5D9CE]'}`}>
                                            <span className="font-bold text-[#5E5049]">{item.name}</span>
                                            <span className="text-[10px] text-[#8C7B6C]">{item.duration}分</span>
                                    </div>
                                ))}
                            </div>
                        </div>
                    ))}
                </div>
                <Button className="w-full" onClick={() => setStep(3)}>下一步</Button>
            </div>
        )}

        {step === 3 && (<div className="space-y-6 animate-fadeIn"><button onClick={() => setStep(2)} className="flex items-center text-[#8C7B6C] mb-2"><ChevronLeft size={16}/> 返回上一步</button><Card className="p-6 space-y-5"><div className="space-y-1"><label className="text-xs font-bold text-[#8C7B6C]">預約日期</label><input type="date" className="w-full p-3 bg-[#F9F7F2] rounded-xl outline-none text-[#5E5049]" value={formData.date} onChange={e => setFormData({...formData, date:e.target.value, staff: null})} /></div><div className="space-y-1"><label className="text-xs font-bold text-[#8C7B6C]">預約時間</label><select className="w-full p-3 bg-[#F9F7F2] rounded-xl outline-none text-[#5E5049]" value={formData.time} onChange={e => setFormData({...formData, time:e.target.value, staff: null})}>
                            <option value="">請選擇時間</option>
                            {settings.timeSlots.map(t => <option key={t} value={t}>{t}</option>)}
                        </select></div></Card>
                        <div className={`transition-all duration-500 ${(!formData.date || !formData.time) ? 'opacity-50 pointer-events-none grayscale' : 'opacity-100'}`}>
                            <h2 className="text-xl font-bold text-[#5E5049] mb-3">選擇服務人員</h2>
                            <div className="grid grid-cols-2 gap-3">
                                <Card onClick={() => setFormData({...formData, staff: null})} className={`p-4 cursor-pointer flex flex-col items-center justify-center gap-2 ${formData.staff === null ? 'border-[#C7B299] ring-1 ring-[#C7B299]' : 'hover:border-[#C7B299]'}`}>
                                    <div className="w-14 h-14 rounded-full bg-[#EBE5DF] flex items-center justify-center text-[#8C7B6C]"><User size={24} /></div>
                                    <div className="font-bold text-[#5E5049]">不指定</div>
                                    <div className="text-xs text-[#8C7B6C]">由店家安排</div>
                                </Card>
                                {staff.map(s => {
                                    const available = isStaffAvailable(s);
                                    return (
                                        <Card key={s.id} 
                                            onClick={() => { if(available) setFormData({...formData, staff: s}) }} 
                                            className={`p-4 flex flex-col items-center justify-center gap-2 relative
                                                ${!available ? 'opacity-60 cursor-not-allowed bg-[#F5F5F5]' : 'cursor-pointer'}
                                                ${formData.staff?.id === s.id ? 'border-[#C7B299] ring-1 ring-[#C7B299]' : 'hover:border-[#C7B299]'}
                                            `}
                                        >
                                            <div className="w-14 h-14 rounded-full bg-[#E5D9CE] flex items-center justify-center text-[#5E5049] font-bold text-lg">{s.name[0]}</div>
                                            <div className="font-bold text-[#5E5049]">{s.name}</div>
                                            <div className="text-xs text-[#8C7B6C]">{s.role}</div>
                                            {!available && (<div className="absolute inset-0 flex items-center justify-center bg-white/70 rounded-2xl backdrop-blur-[1px]"><span className="bg-[#5E5049] text-white text-xs px-2 py-1 rounded shadow-sm">未開放</span></div>)}
                                        </Card>
                                    );
                                })}
                            </div>
                        </div>

                        <Button className="w-full shadow-lg" onClick={() => setStep(4)} disabled={!formData.date || !formData.time}>下一步</Button>
                    </div>
        )}
        {step === 4 && (<div className="space-y-6 animate-fadeIn"><button onClick={() => setStep(3)} className="flex items-center text-[#8C7B6C] mb-2"><ChevronLeft size={16}/> 返回上一步</button><Card className="p-6 space-y-5"><div className="pt-4 border-t border-[#E5D9CE] space-y-4"><div className="space-y-1"><label className="text-sm font-bold text-[#8C7B6C] flex gap-1">您的姓名 <span className="text-red-500">*</span></label><input placeholder="請輸入姓名" className="w-full p-3 bg-[#F9F7F2] rounded-xl outline-none" value={formData.name} onChange={e=>setFormData({...formData, name:e.target.value})} /></div><div className="space-y-1"><label className="text-sm font-bold text-[#8C7B6C] flex gap-1">聯絡電話 <span className="text-red-500">*</span></label><input placeholder="請輸入電話" className="w-full p-3 bg-[#F9F7F2] rounded-xl outline-none" value={formData.phone} onChange={handlePhoneChange} /></div><div className="space-y-1"><label className="text-sm font-bold text-[#8C7B6C] flex gap-1">LINE ID / 暱稱</label><input placeholder="選填" className="w-full p-3 bg-[#F9F7F2] rounded-xl outline-none" value={formData.lineId} onChange={e=>setFormData({...formData, lineId:e.target.value})} /></div><div className="space-y-1"><label className="text-sm font-bold text-[#8C7B6C]">備註需求</label><textarea placeholder="選填" className="w-full p-3 bg-[#F9F7F2] rounded-xl outline-none resize-none h-20" value={formData.notes} onChange={e=>setFormData({...formData, notes:e.target.value})} /></div></div><div className="bg-[#FDFBF7] p-4 rounded-xl border border-[#E5D9CE] space-y-2"><h4 className="font-bold text-[#5E5049] text-sm">預約須知</h4><p className="text-xs text-[#8C7B6C] whitespace-pre-wrap">{settings.reservationRules}</p><label className="flex items-center gap-2 pt-2 border-t border-[#E5D9CE] cursor-pointer"><input type="checkbox" className="w-4 h-4 accent-[#C7B299]" checked={agreedToRules} onChange={(e) => setAgreedToRules(e.target.checked)} /><span className="text-sm text-[#5E5049] font-bold">我已閱讀並同意</span></label></div></Card><div className="flex justify-between items-center px-2"><span className="text-[#8C7B6C]">總金額</span><span className="text-2xl font-bold text-[#C7B299]">${calculateTotal()}</span></div><Button className="w-full shadow-lg" onClick={handleSubmit} disabled={!formData.date || !formData.time || !formData.name || !formData.phone || !agreedToRules}>確認預約</Button></div>)}
    </div>
  );
}
