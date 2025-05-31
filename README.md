import React, { useState, useEffect, useMemo, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithEmailAndPassword, createUserWithEmailAndPassword, onAuthStateChanged, signOut, GoogleAuthProvider, signInWithPopup } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, where, updateDoc, doc, deleteDoc } from 'firebase/firestore';
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer, PieChart, Pie, Cell } from 'recharts';

// Initialize Firebase config from global variables
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// Initialize Firebase App
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// List of Moroccan cities (retained for potential future use or other forms)
const MOROCCAN_CITIES = [
  'الدار البيضاء', 'الرباط', 'فاس', 'مراكش', 'أكادير', 'طنجة', 'مكناس', 'وجدة',
  'القنيطرة', 'تطوان', 'أسفي', 'سلا', 'بني ملال', 'خريبكة', 'الجديدة', 'الناظور',
  'تازة', 'العيون', 'الرشيدية', 'ورزازات', 'الصويرة', 'شفشاون', 'الحسيمة', 'كلميم',
  'الداخلة', 'بركان', 'سطات', 'بني انصار', 'جرادة', 'تزنيت'
].sort();

// Reusable Modal Component
const Modal = ({ isOpen, onClose, title, children }) => {
  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
      <div className="bg-white rounded-lg shadow-xl w-full max-w-md mx-auto p-6 relative">
        <h3 className="text-2xl font-bold text-indigo-700 mb-4 border-b pb-2">{title}</h3>
        <button
          onClick={onClose}
          className="absolute top-4 right-4 text-gray-500 hover:text-gray-700 text-2xl font-bold"
        >
          &times;
        </button>
        <div className="max-h-96 overflow-y-auto pr-2">
          {children}
        </div>
      </div>
    </div>
  );
};

// Reusable Toast Notification Component
const Toast = ({ message, type, onClose }) => {
  const bgColor = type === 'success' ? 'bg-green-500' : 'bg-red-500';
  const textColor = 'text-white';

  useEffect(() => {
    const timer = setTimeout(() => {
      onClose();
    }, 3000);
    return () => clearTimeout(timer);
  }, [onClose]);

  return (
    <div className={`fixed bottom-4 right-4 p-4 rounded-lg shadow-lg ${bgColor} ${textColor} z-50 transition-transform transform duration-300 ease-out translate-y-0 opacity-100`}>
      {message}
    </div>
  );
};

// Simple Base64 encoding/decoding for demonstration.
// WARNING: This is NOT secure encryption for sensitive data like passwords.
// A real application requires strong cryptographic methods and secure key management.
const encodeBase64 = (str) => btoa(str);
const decodeBase64 = (str) => atob(str);

function App() {
  const [user, setUser] = useState(null);
  const [loadingAuth, setLoadingAuth] = useState(true);
  const [view, setView] = useState('login'); // 'login', 'register', 'dashboard', 'passwordManager', 'genericDashboard'
  const [toastMessage, setToastMessage] = useState(null);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      setLoadingAuth(false);
      if (currentUser && (view === 'login' || view === 'register')) {
        setView('dashboard'); // Redirect to dashboard if logged in
      } else if (!currentUser && view !== 'register') {
        setView('login'); // Redirect to login if logged out
      }
    });
    return () => unsubscribe();
  }, [view]);

  const handleLogout = async () => {
    try {
      await signOut(auth);
      setToastMessage({ message: 'تم تسجيل الخروج بنجاح!', type: 'success' });
      setView('login');
    } catch (error) {
      setToastMessage({ message: 'فشل تسجيل الخروج: ' + error.message, type: 'error' });
    }
  };

  if (loadingAuth) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-100 font-inter">
        <div className="text-2xl text-gray-700">جاري التحميل...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 font-inter text-gray-800 p-4 sm:p-6 lg:p-8">
      <div className="max-w-7xl mx-auto bg-white rounded-xl shadow-lg p-6 sm:p-8">
        {/* Header and Navigation */}
        <header className="mb-8 border-b pb-4 flex flex-col sm:flex-row justify-between items-center">
          <h1 className="text-3xl sm:text-4xl font-extrabold text-indigo-700 mb-4 sm:mb-0">
            منصة بياناتك الآمنة
          </h1>
          {user ? (
            <div className="flex flex-wrap justify-center sm:justify-end gap-2 sm:space-x-4">
              <button
                onClick={() => setView('dashboard')}
                className={`px-4 py-2 rounded-lg font-semibold transition duration-300 ${
                  view === 'dashboard' ? 'bg-indigo-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-indigo-100'
                }`}
              >
                الرئيسية
              </button>
              <button
                onClick={() => setView('passwordManager')}
                className={`px-4 py-2 rounded-lg font-semibold transition duration-300 ${
                  view === 'passwordManager' ? 'bg-indigo-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-indigo-100'
                }`}
              >
                إدارة كلمات المرور
              </button>
              <button
                onClick={() => setView('genericDashboard')}
                className={`px-4 py-2 rounded-lg font-semibold transition duration-300 ${
                  view === 'genericDashboard' ? 'bg-indigo-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-indigo-100'
                }`}
              >
                لوحات البيانات
              </button>
              <button
                onClick={handleLogout}
                className="px-4 py-2 rounded-lg font-semibold transition duration-300 bg-red-600 text-white hover:bg-red-700 shadow-md"
              >
                تسجيل الخروج
              </button>
            </div>
          ) : (
            <div className="flex flex-wrap justify-center sm:justify-end gap-2 sm:space-x-4">
              <button
                onClick={() => setView('login')}
                className={`px-4 py-2 rounded-lg font-semibold transition duration-300 ${
                  view === 'login' ? 'bg-indigo-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-indigo-100'
                }`}
              >
                تسجيل الدخول
              </button>
              <button
                onClick={() => setView('register')}
                className={`px-4 py-2 rounded-lg font-semibold transition duration-300 ${
                  view === 'register' ? 'bg-indigo-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-indigo-100'
                }`}
              >
                إنشاء حساب
              </button>
            </div>
          )}
        </header>

        {user && (
          <div className="text-sm text-gray-500 mb-6 text-right">
            مرحباً، <span className="font-semibold text-indigo-600">{user.email || user.displayName || 'مستخدم'}</span>! معرف المستخدم: <span className="font-mono text-gray-600">{user.uid}</span>
          </div>
        )}

        {/* Main Content Area */}
        {!user && view === 'login' && <Login setView={setView} setToastMessage={setToastMessage} />}
        {!user && view === 'register' && <Register setView={setView} setToastMessage={setToastMessage} />}
        {user && view === 'dashboard' && <Dashboard user={user} setToastMessage={setToastMessage} />}
        {user && view === 'passwordManager' && <PasswordManager user={user} setToastMessage={setToastMessage} />}
        {user && view === 'genericDashboard' && <GenericDashboard user={user} setToastMessage={setToastMessage} />}

        {/* Toast Notification */}
        {toastMessage && (
          <Toast
            message={toastMessage.message}
            type={toastMessage.type}
            onClose={() => setToastMessage(null)}
          />
        )}
      </div>
    </div>
  );
}

// Login Component
function Login({ setView, setToastMessage }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      await signInWithEmailAndPassword(auth, email, password);
      setToastMessage({ message: 'تم تسجيل الدخول بنجاح!', type: 'success' });
      setEmail('');
      setPassword('');
      setView('dashboard');
    } catch (error) {
      setToastMessage({ message: 'فشل تسجيل الدخول: ' + error.message, type: 'error' });
    } finally {
      setLoading(false);
    }
  };

  const handleGoogleLogin = async () => {
    setLoading(true);
    try {
      const provider = new GoogleAuthProvider();
      await signInWithPopup(auth, provider);
      setToastMessage({ message: 'تم تسجيل الدخول بحساب Google بنجاح!', type: 'success' });
      setView('dashboard');
    } catch (error) {
      setToastMessage({ message: 'فشل تسجيل الدخول بحساب Google: ' + error.message, type: 'error' });
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="p-6 bg-white rounded-xl shadow-lg max-w-md mx-auto">
      <h2 className="text-2xl font-bold text-indigo-700 mb-6 text-center">تسجيل الدخول</h2>
      <form onSubmit={handleLogin} className="space-y-4">
        <div>
          <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-1">البريد الإلكتروني</label>
          <input
            type="email"
            id="email"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="password" className="block text-sm font-medium text-gray-700 mb-1">كلمة المرور</label>
          <input
            type="password"
            id="password"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>
        <button
          type="submit"
          className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 transform hover:scale-105"
          disabled={loading}
        >
          {loading ? 'جاري تسجيل الدخول...' : 'تسجيل الدخول'}
        </button>
      </form>
      <div className="mt-4 text-center">
        <button
          onClick={handleGoogleLogin}
          className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 transform hover:scale-105 flex items-center justify-center gap-2"
          disabled={loading}
        >
          <svg className="w-5 h-5" viewBox="0 0 24 24" fill="currentColor" xmlns="http://www.w3.org/2000/svg">
            <path d="M12.24 10.29v3.51h6.41c-.29 1.62-1.25 3.34-3.13 4.54l-.06.05-2.73 2.12c-1.38 1.08-3.1 1.69-5.02 1.69-3.87 0-7.02-3.15-7.02-7.02s3.15-7.02 7.02-7.02c2.1 0 3.93.76 5.37 2.12l2.36-2.36c-1.74-1.63-4.01-2.64-6.73-2.64-5.78 0-10.48 4.7-10.48 10.48s4.7 10.48 10.48 10.48c5.78 0 10.48-4.7 10.48-10.48 0-.8-.09-1.57-.26-2.32h-10.22z" fill="#4285F4"/>
            <path d="M23.01 12.02c0-.8-.09-1.57-.26-2.32h-10.22v3.51h6.41c-.29 1.62-1.25 3.34-3.13 4.54l-.06.05-2.73 2.12c-1.38 1.08-3.1 1.69-5.02 1.69-3.87 0-7.02-3.15-7.02-7.02s3.15-7.02 7.02-7.02c2.1 0 3.93.76 5.37 2.12l2.36-2.36c-1.74-1.63-4.01-2.64-6.73-2.64-5.78 0-10.48 4.7-10.48 10.48s4.7 10.48 10.48 10.48c5.78 0 10.48-4.7 10.48-10.48 0-.8-.09-1.57-.26-2.32z" fill="#34A853"/>
            <path d="M12.24 2.02c1.88 0 3.5.65 4.79 1.77l2.36-2.36c-1.63-1.53-3.8-2.43-6.15-2.43-3.87 0-7.02 3.15-7.02 7.02s3.15 7.02 7.02 7.02c2.1 0 3.93.76 5.37 2.12l2.36-2.36c-1.74-1.63-4.01-2.64-6.73-2.64-5.78 0-10.48 4.7-10.48 10.48s4.7 10.48 10.48 10.48c5.78 0 10.48-4.7 10.48-10.48 0-.8-.09-1.57-.26-2.32z" fill="#FBBC05"/>
            <path d="M12.24 22.02c-1.88 0-3.5-.65-4.79-1.77l-2.36 2.36c1.63 1.53 3.8 2.43 6.15 2.43 3.87 0 7.02-3.15 7.02-7.02s-3.15-7.02-7.02-7.02c-2.1 0-3.93-.76-5.37-2.12l-2.36 2.36c1.74 1.63 4.01 2.64 6.73 2.64 5.78 0 10.48-4.7 10.48-10.48s-4.7-10.48-10.48-10.48 0-.8-.09-1.57-.26-2.32z" fill="#EA4335"/>
          </svg>
          تسجيل الدخول باستخدام Google
        </button>
      </div>
      <p className="mt-4 text-center text-gray-600">
        ليس لديك حساب؟{' '}
        <button onClick={() => setView('register')} className="text-indigo-600 hover:underline font-semibold">
          إنشاء حساب جديد
        </button>
      </p>
    </div>
  );
}

// Register Component
function Register({ setView, setToastMessage }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [name, setName] = useState('');
  const [loading, setLoading] = useState(false);

  const handleRegister = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      const userCredential = await createUserWithEmailAndPassword(auth, email, password);
      const user = userCredential.user;

      // Simulate sending data to Google Sheets
      console.log(`Simulating sending new customer data to Google Sheets:
        Email: ${email},
        Name: ${name},
        UserID: ${user.uid}
        (In a real app, this would be handled by a secure backend API call to Google Sheets)
      `);
      setToastMessage({ message: 'تم إنشاء الحساب بنجاح! (تمت محاكاة إرسال البيانات إلى Google Sheets)', type: 'success' });
      setEmail('');
      setPassword('');
      setName('');
      setView('dashboard');
    } catch (error) {
      setToastMessage({ message: 'فشل إنشاء الحساب: ' + error.message, type: 'error' });
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="p-6 bg-white rounded-xl shadow-lg max-w-md mx-auto">
      <h2 className="text-2xl font-bold text-indigo-700 mb-6 text-center">إنشاء حساب جديد</h2>
      <form onSubmit={handleRegister} className="space-y-4">
        <div>
          <label htmlFor="register-name" className="block text-sm font-medium text-gray-700 mb-1">الاسم</label>
          <input
            type="text"
            id="register-name"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={name}
            onChange={(e) => setName(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="register-email" className="block text-sm font-medium text-gray-700 mb-1">البريد الإلكتروني</label>
          <input
            type="email"
            id="register-email"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="register-password" className="block text-sm font-medium text-gray-700 mb-1">كلمة المرور</label>
          <input
            type="password"
            id="register-password"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>
        <button
          type="submit"
          className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 transform hover:scale-105"
          disabled={loading}
        >
          {loading ? 'جاري إنشاء الحساب...' : 'إنشاء حساب'}
        </button>
      </form>
      <p className="mt-4 text-center text-gray-600">
        لديك حساب بالفعل؟{' '}
        <button onClick={() => setView('login')} className="text-indigo-600 hover:underline font-semibold">
          تسجيل الدخول
        </button>
      </p>
    </div>
  );
}

// Main Dashboard Component (Simplified for generic data)
function Dashboard({ user, setToastMessage }) {
  return (
    <div className="p-6 bg-white rounded-xl shadow-lg">
      <h2 className="text-2xl font-bold text-indigo-700 mb-6">لوحة التحكم الرئيسية</h2>
      <p className="text-gray-700 mb-4">
        مرحباً بك في منصة بياناتك الآمنة، {user.email || user.displayName || 'مستخدم'}!
      </p>
      <p className="text-gray-600">
        هنا يمكنك الوصول إلى لوحات بياناتك المخصصة وإدارة كلمات المرور الخاصة بك بأمان.
      </p>

      <div className="mt-8 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        <div className="bg-blue-500 text-white p-6 rounded-xl shadow-lg flex flex-col items-center justify-center text-center">
          <span className="text-6xl mb-3">🔑</span>
          <h3 className="text-xl font-bold">إدارة كلمات المرور</h3>
          <p className="text-sm opacity-90 mt-2">احفظ وادير كلمات مرورك بأمان.</p>
        </div>
        <div className="bg-green-500 text-white p-6 rounded-xl shadow-lg flex flex-col items-center justify-center text-center">
          <span className="text-6xl mb-3">📊</span>
          <h3 className="text-xl font-bold">لوحات البيانات المخصصة</h3>
          <p className="text-sm opacity-90 mt-2">تتبع بياناتك الهامة من مكان واحد.</p>
        </div>
        <div className="bg-purple-500 text-white p-6 rounded-xl shadow-lg flex flex-col items-center justify-center text-center">
          <span className="text-6xl mb-3">🚀</span>
          <h3 className="text-xl font-bold">ميزات قادمة</h3>
          <p className="text-sm opacity-90 mt-2">نعمل باستمرار على إضافة المزيد من الأدوات القوية.</p>
        </div>
      </div>
    </div>
  );
}

// Password Manager Component
function PasswordManager({ user, setToastMessage }) {
  const [passwords, setPasswords] = useState([]);
  const [website, setWebsite] = useState('');
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [showPassword, setShowPassword] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [loading, setLoading] = useState(false);

  // Fetch passwords specific to the logged-in user
  useEffect(() => {
    if (!user || !user.uid) return;

    setLoading(true);
    const passwordsColRef = collection(db, `artifacts/${appId}/users/${user.uid}/passwords`);
    const unsubscribe = onSnapshot(passwordsColRef, (snapshot) => {
      const fetchedPasswords = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setPasswords(fetchedPasswords);
      setLoading(false);
    }, (err) => {
      console.error("Error fetching passwords:", err);
      setToastMessage({ message: 'فشل جلب كلمات المرور: ' + err.message, type: 'error' });
      setLoading(false);
    });

    return () => unsubscribe();
  }, [user, setToastMessage]);

  const handleAddOrUpdatePassword = async (e) => {
    e.preventDefault();
    if (!website || !username || !password) {
      setToastMessage({ message: 'يرجى ملء جميع الحقول المطلوبة.', type: 'error' });
      return;
    }

    setLoading(true);
    try {
      const encryptedPassword = encodeBase64(password); // Simple encoding for demo
      const passwordData = { website, username, password: encryptedPassword };

      if (editingId) {
        await updateDoc(doc(db, `artifacts/${appId}/users/${user.uid}/passwords`, editingId), passwordData);
        setToastMessage({ message: 'تم تحديث كلمة المرور بنجاح!', type: 'success' });
      } else {
        await addDoc(collection(db, `artifacts/${appId}/users/${user.uid}/passwords`), passwordData);
        setToastMessage({ message: 'تم إضافة كلمة المرور بنجاح!', type: 'success' });
      }
      setWebsite('');
      setUsername('');
      setPassword('');
      setEditingId(null);
    } catch (error) {
      setToastMessage({ message: 'فشل حفظ كلمة المرور: ' + error.message, type: 'error' });
    } finally {
      setLoading(false);
    }
  };

  const handleEdit = (pwd) => {
    setEditingId(pwd.id);
    setWebsite(pwd.website);
    setUsername(pwd.username);
    setPassword(decodeBase64(pwd.password)); // Decode for editing
  };

  const handleDelete = async (id) => {
    if (!window.confirm('هل أنت متأكد أنك تريد حذف كلمة المرور هذه؟')) {
      return;
    }
    setLoading(true);
    try {
      await deleteDoc(doc(db, `artifacts/${appId}/users/${user.uid}/passwords`, id));
      setToastMessage({ message: 'تم حذف كلمة المرور بنجاح!', type: 'success' });
    } catch (error) {
      setToastMessage({ message: 'فشل حذف كلمة المرور: ' + error.message, type: 'error' });
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="p-6 bg-white rounded-xl shadow-lg">
      <h2 className="text-2xl font-bold text-indigo-700 mb-6">إدارة كلمات المرور</h2>

      {/* Add/Edit Password Form */}
      <form onSubmit={handleAddOrUpdatePassword} className="space-y-4 mb-8 p-4 border border-gray-200 rounded-lg shadow-sm">
        <h3 className="text-xl font-semibold text-gray-800 mb-3">{editingId ? 'تعديل كلمة المرور' : 'إضافة كلمة مرور جديدة'}</h3>
        <div>
          <label htmlFor="website" className="block text-sm font-medium text-gray-700 mb-1">الموقع/الخدمة</label>
          <input
            type="text"
            id="website"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={website}
            onChange={(e) => setWebsite(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="username" className="block text-sm font-medium text-gray-700 mb-1">اسم المستخدم/البريد الإلكتروني</label>
          <input
            type="text"
            id="username"
            className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            value={username}
            onChange={(e) => setUsername(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="password" className="block text-sm font-medium text-gray-700 mb-1">كلمة المرور</label>
          <div className="relative">
            <input
              type={showPassword ? 'text' : 'password'}
              id="password"
              className="mt-1 block w-full p-3 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500 pr-10"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
            />
            <button
              type="button"
              onClick={() => setShowPassword(!showPassword)}
              className="absolute inset-y-0 right-0 pr-3 flex items-center text-sm leading-5"
            >
              {showPassword ? '👁️‍🗨️' : '🔒'}
            </button>
          </div>
        </div>
        <button
          type="submit"
          className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-300 transform hover:scale-105"
          disabled={loading}
        >
          {loading ? 'جاري الحفظ...' : (editingId ? 'تعديل كلمة المرور' : 'إضافة كلمة المرور')}
        </button>
        {editingId && (
          <button
            type="button"
            onClick={() => { setEditingId(null); setWebsite(''); setUsername(''); setPassword(''); }}
            className="w-full mt-2 bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-3 px-4 rounded-lg shadow-md transition duration-300"
          >
            إلغاء التعديل
          </button>
        )}
      </form>

      {/* Passwords List */}
      <h3 className="text-xl font-semibold text-gray-800 mb-3">كلمات المرور المحفوظة ({passwords.length})</h3>
      {loading ? (
        <div className="text-center text-gray-600">جاري تحميل كلمات المرور...</div>
      ) : passwords.length === 0 ? (
        <p className="text-center text-gray-500">لا توجد كلمات مرور محفوظة بعد.</p>
      ) : (
        <div className="overflow-x-auto">
          <table className="min-w-full bg-white rounded-lg overflow-hidden">
            <thead className="bg-indigo-500 text-white">
              <tr>
                <th className="py-3 px-4 text-right">الموقع/الخدمة</th>
                <th className="py-3 px-4 text-right">اسم المستخدم</th>
                <th className="py-3 px-4 text-right">كلمة المرور</th>
                <th className="py-3 px-4 text-right">الإجراءات</th>
              </tr>
            </thead>
            <tbody>
              {passwords.map((pwd, index) => (
                <tr key={pwd.id} className={`border-b ${index % 2 === 0 ? 'bg-gray-50' : 'bg-white'}`}>
                  <td className="py-3 px-4 font-medium">{pwd.website}</td>
                  <td className="py-3 px-4">{pwd.username}</td>
                  <td className="py-3 px-4">
                    <span className="font-mono text-sm">
                      {showPassword ? decodeBase64(pwd.password) : '********'}
                    </span>
                  </td>
                  <td className="py-3 px-4 flex gap-2">
                    <button
                      onClick={() => handleEdit(pwd)}
                      className="px-3 py-1 bg-blue-500 text-white rounded-md text-sm hover:bg-blue-600 transition duration-300"
                    >
                      تعديل
                    </button>
                    <button
                      onClick={() => handleDelete(pwd.id)}
                      className="px-3 py-1 bg-red-500 text-white rounded-md text-sm hover:bg-red-600 transition duration-300"
                    >
                      حذف
                    </button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
      <div className="mt-4 text-center">
        <button
          onClick={() => setShowPassword(!showPassword)}
          className="px-4 py-2 bg-gray-200 text-gray-700 rounded-lg font-semibold hover:bg-gray-300 transition duration-300"
        >
          {showPassword ? 'إخفاء كلمات المرور' : 'إظهار كلمات المرور'}
        </button>
      </div>
    </div>
  );
}

// Generic Dashboard Component (Placeholder)
function GenericDashboard({ user, setToastMessage }) {
  return (
    <div className="p-6 bg-white rounded-xl shadow-lg">
      <h2 className="text-2xl font-bold text-indigo-700 mb-6">لوحات البيانات المخصصة</h2>
      <p className="text-gray-700 mb-4">
        هذه مساحة مخصصة للوحات البيانات الخاصة بك، {user.email || user.displayName || 'مستخدم'}!
      </p>
      <p className="text-gray-600">
        يمكنك هنا عرض تحليلات المبيعات، تتبع المخزون، أو أي بيانات أخرى تهمك.
        في المستقبل، يمكننا إضافة أدوات لربط مصادر البيانات المختلفة وإنشاء تصورات بيانية ديناميكية.
      </p>
      <div className="mt-8 p-6 bg-gray-100 rounded-lg text-center border border-dashed border-gray-300">
        <p className="text-gray-500 text-lg">
          لا توجد لوحات بيانات مخصصة بعد.
        </p>
        <p className="text-gray-400 text-sm mt-2">
          (هذه الميزة قيد التطوير ويمكن تخصيصها لاحقًا.)
        </p>
        <ResponsiveContainer width="100%" height={200} className="mt-4">
          <BarChart data={[{name: 'بيانات', value: 10}, {name: 'أخرى', value: 20}]}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="name" />
            <YAxis />
            <Tooltip />
            <Legend />
            <Bar dataKey="value" fill="#8884d8" />
          </BarChart>
        </ResponsiveContainer>
      </div>
    </div>
  );
}

export default App;
