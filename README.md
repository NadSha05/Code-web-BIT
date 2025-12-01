import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged,
  signOut, // Tambahan: Fungsi Logout
  signInWithEmailAndPassword, // Tambahan: Login Email/Password
  createUserWithEmailAndPassword // Tambahan: Register Email/Password
} from 'firebase/auth';
import {
  getFirestore,
  collection,
  onSnapshot,
  addDoc,
  query,
  serverTimestamp,
  setLogLevel,
  doc,
  setDoc,
  runTransaction,
  getDocs // Untuk mengambil semua reviews (Publik)
} from 'firebase/firestore';
import { 
  Truck, MapPin, Recycle, Trash2, Clock, User, Save, Phone, 
  DollarSign, Package, CheckCircle, Gift, BookOpen, Clock3, Loader2, Star, LogOut, Mail, Lock 
} from 'lucide-react';

// Variabel Global yang Disediakan Lingkungan Canvas
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Mengatur level log Firebase ke Debug
setLogLevel('debug');

// --- DATA MOCK (Untuk Simulasi Fitur Baru: Bank Sampah, Reward, Edukasi) ---
const MOCK_BANK_SAMPAH = [
  { name: "BS Mandiri Jaya", distance: "1.2 km", address: "Jl. Sudirman No. 12, Jakarta" },
  { name: "BS Eco Hijau", distance: "3.5 km", address: "Komplek Hijau Lestari No. 5" },
  { name: "Pusat Daur Ulang Kota", distance: "5.1 km", address: "Gedung Pusat Daur Ulang" },
];

const MOCK_REWARDS = [
    { id: 1, name: "Voucher Kopi", points: 5000, description: "Diskon 50% di kafe mitra." },
    { id: 2, name: "Pulsa 25K", points: 25000, description: "Pulsa operator seluler senilai Rp25.000." },
    { id: 3, name: "E-Wallet 50K", points: 50000, description: "Saldo E-Wallet senilai Rp50.000." },
];

const MOCK_ARTICLES = [
    { title: "Panduan Memilah Sampah Plastik", source: "EcoEdu", date: "10 Nov 2025" },
    { title: "Bahaya E-Waste dan Penanganannya", source: "Kementerian LHK", date: "01 Sep 2025" },
    { title: "Kompos Sederhana di Rumah", source: "GreenTips", date: "22 Okt 2025" },
];


// --- UTILITY COMPONENTS ---

// Komponen Kartu KPI (Key Performance Indicator)
const KPICard = ({ title, value, unit, color, icon: Icon }) => (
  <div className={`p-4 rounded-xl shadow-lg transform hover:scale-[1.02] transition duration-300 ${color} flex flex-col justify-between`}>
    <div className="flex items-center justify-between">
      <h3 className="text-sm font-medium text-white opacity-90">{title}</h3>
      <Icon className="w-6 h-6 text-white opacity-80" />
    </div>
    <div className="mt-4">
      <p className="text-3xl font-bold text-white leading-none">
        {value}
      </p>
      <p className="text-xs text-white opacity-70 mt-1">{unit}</p>
    </div>
  </div>
);

// Komponen Rating Bintang
const StarRating = ({ rating, size = 5 }) => {
    return (
        <div className="flex items-center">
            {[...Array(5)].map((_, i) => (
                <Star
                    key={i}
                    className={`w-${size} h-${size} ${i < rating ? 'text-yellow-400 fill-yellow-400' : 'text-gray-300'}`}
                />
            ))}
        </div>
    );
};


// --- MAIN APP COMPONENT ---

function App() {
  // State untuk Firebase dan Autentikasi
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [isLoggedIn, setIsLoggedIn] = useState(false); // Tambahan: Status Login
  const [isAnonymous, setIsAnonymous] = useState(false);

  // State Navigasi
  const [activeTab, setActiveTab] = useState('dashboard'); // 'dashboard', 'tracking', 'ecosystem', 'review'

  // State Profil Pengguna
  const [userName, setUserName] = useState('');
  const [userPhone, setUserPhone] = useState('');
  const [currentName, setCurrentName] = useState('Anonim');
  const [profileMessage, setProfileMessage] = useState('');
  const [isProfileLoading, setIsProfileLoading] = useState(false);

  // State Aplikasi (Riwayat Penjemputan)
  const [pickups, setPickups] = useState([]);
  const [uiMessage, setUiMessage] = useState('');
  
  // State Metrik Pengguna (KPI - Diambil dari profil real-time)
  const [totalWeight, setTotalWeight] = useState(0);
  const [totalPoints, setTotalPoints] = useState(0);
  const [completedPickupsCount, setCompletedPickupsCount] = useState(0);
  
  // State Penjemputan Baru (untuk form Penjemputan Khusus)
  const [newLocation, setNewLocation] = useState('');
  const [newWeight, setNewWeight] = useState(0);
  const [newWasteType, setNewWasteType] = useState('E-Waste');
  const [isPickupLoading, setIsPickupLoading] = useState(false);
  const [pickupUiMessage, setPickupUiMessage] = useState('');

  // State Review
  const [allReviews, setAllReviews] = useState([]);
  const [newRating, setNewRating] = useState(0);
  const [newReviewText, setNewReviewText] = useState('');
  const [reviewMessage, setReviewMessage] = useState('');
  const [isReviewLoading, setIsReviewLoading] = useState(false);

  // State Login/Register
  const [authEmail, setAuthEmail] = useState('');
  const [authPassword, setAuthPassword] = useState('');
  const [authMessage, setAuthMessage] = useState('');
  const [isAuthLoading, setIsAuthLoading] = useState(false);
  const [isRegisterMode, setIsRegisterMode] = useState(false);


  // 1. Inisialisasi Firebase dan Autentikasi
  useEffect(() => {
    try {
      if (Object.keys(firebaseConfig).length === 0) {
        console.error("Konfigurasi Firebase tidak tersedia.");
        return;
      }
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const userAuth = getAuth(app);
      setDb(firestore);
      setAuth(userAuth);

      const unsubscribe = onAuthStateChanged(userAuth, async (user) => {
        let currentUserId = userAuth.currentUser?.uid;
        let isAnon = false;
        
        if (!user) {
          try {
            if (initialAuthToken) {
                await signInWithCustomToken(userAuth, initialAuthToken);
            } else {
                await signInAnonymously(userAuth);
                isAnon = true;
            }
            currentUserId = userAuth.currentUser?.uid;
          } catch (error) {
            console.error("Gagal Autentikasi, masuk anonim:", error);
            await signInAnonymously(userAuth);
            currentUserId = userAuth.currentUser?.uid;
            isAnon = true;
          }
        } else {
            // Pengguna sudah login (email/custom token)
            isAnon = user.isAnonymous;
        }

        const finalUserId = currentUserId || crypto.randomUUID();
        setUserId(finalUserId);
        setIsAuthReady(true);
        setIsLoggedIn(!isAnon); // Dianggap Login jika bukan Anonim
        setIsAnonymous(isAnon);
        console.log("Autentikasi Firebase siap. User ID:", finalUserId, "Anonim:", isAnon);
      });

      return () => unsubscribe();
    } catch (e) {
      console.error("Gagal inisialisasi Firebase:", e);
    }
  }, []);

  // Helper untuk mendapatkan referensi dokumen profil
  const getProfileDocRef = useCallback(() => {
    if (!db || !userId) return null;
    return doc(db, `artifacts/${appId}/users/${userId}/profile`, 'user_data');
  }, [db, userId]);
  
  // Helper untuk mendapatkan referensi koleksi review publik
  const getPublicReviewCollectionRef = useCallback(() => {
    if (!db) return null;
    // Disimpan di koleksi publik untuk dapat diakses oleh semua
    return collection(db, `artifacts/${appId}/public/data/reviews`);
  }, [db]);


  // 2. Muat dan Dengarkan Perubahan Profil Pengguna & KPI
  useEffect(() => {
    if (!db || !userId || !isAuthReady) return;

    const profileRef = getProfileDocRef();
    if (!profileRef) return;

    // Mendengarkan perubahan profil secara real-time
    const unsubscribeProfile = onSnapshot(profileRef, (docSnap) => {
        if (docSnap.exists()) {
            const data = docSnap.data();
            setUserName(data.name || '');
            setUserPhone(data.phone || '');
            setCurrentName(data.name || (isAnonymous ? 'Pengguna Anonim' : docSnap.data().email || 'Pengguna Terdaftar'));
            // Data KPI (Total Weight & Points)
            setTotalWeight(data.totalWeight || 0);
            setTotalPoints(data.totalPoints || 0);
            setCompletedPickupsCount(data.completedPickupsCount || 0);
            console.log("Profil pengguna real-time dimuat/diperbarui.");
        } else {
            // Jika dokumen belum ada, buat dokumen awal
            setDoc(profileRef, {
                name: isAnonymous ? 'Pengguna Anonim' : auth?.currentUser?.email || 'Pengguna Baru',
                email: auth?.currentUser?.email || null,
                phone: '',
                totalWeight: 0,
                totalPoints: 0,
                completedPickupsCount: 0,
                createdAt: serverTimestamp()
            }, { merge: true }).catch(e => console.error("Gagal membuat profil awal:", e));
        }
    }, (error) => {
        console.error("Gagal mendengarkan profil:", error);
    });

    return () => unsubscribeProfile();
  }, [db, userId, isAuthReady, getProfileDocRef, isAnonymous, auth]);
  
  // 3. Ambil Data Penjemputan (Real-time)
  useEffect(() => {
    if (!db || !userId || !isAuthReady) return;

    try {
      const pickupCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/pickup_requests`);
      const q = query(pickupCollectionRef);

      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedPickups = snapshot.docs.map(doc => ({
          id: doc.id,
          ...doc.data(),
        }));
        fetchedPickups.sort((a, b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0));
        setPickups(fetchedPickups);
      }, (error) => {
        console.error("Gagal mendengarkan perubahan koleksi penjemputan:", error);
        setUiMessage(`Error memuat data: ${error.message}`);
      });

      return () => unsubscribe();
    } catch (e) {
      console.error("Gagal mengatur listener Firestore:", e);
    }
  }, [db, userId, isAuthReady]);

  // 4. Ambil Data Review Publik (Real-time)
  useEffect(() => {
    const reviewCollectionRef = getPublicReviewCollectionRef();
    if (!db || !reviewCollectionRef) return;
    
    const unsubscribeReviews = onSnapshot(reviewCollectionRef, (snapshot) => {
        const fetchedReviews = snapshot.docs.map(doc => ({
            id: doc.id,
            ...doc.data(),
        }));
        fetchedReviews.sort((a, b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0));
        setAllReviews(fetchedReviews);
    }, (error) => {
        console.error("Gagal mendengarkan review publik:", error);
    });

    return () => unsubscribeReviews();
  }, [db, getPublicReviewCollectionRef]);


  // --- HANDLERS AUTHENTIKASI ---

  const handleAuth = async (e) => {
    e.preventDefault();
    if (!auth || !db) return;

    if (!authEmail || authPassword.length < 6) {
        setAuthMessage("Email dan password (min 6 karakter) harus diisi.");
        return;
    }
    
    setIsAuthLoading(true);
    setAuthMessage('');

    try {
        if (isRegisterMode) {
            // REGISTER
            const userCredential = await createUserWithEmailAndPassword(auth, authEmail, authPassword);
            const user = userCredential.user;
            // Buat dokumen profil awal
            await setDoc(doc(db, `artifacts/${appId}/users/${user.uid}/profile`, 'user_data'), {
                name: user.email.split('@')[0],
                email: user.email,
                totalWeight: 0,
                totalPoints: 0,
                completedPickupsCount: 0,
                createdAt: serverTimestamp()
            }, { merge: true });
            setAuthMessage("Pendaftaran berhasil! Silakan masuk.");
            setIsRegisterMode(false); // Kembali ke mode login setelah daftar
        } else {
            // LOGIN
            await signInWithEmailAndPassword(auth, authEmail, authPassword);
            setAuthMessage("Login berhasil! Memuat dashboard...");
            setIsLoggedIn(true);
        }
    } catch (error) {
        let message = "Terjadi kesalahan autentikasi.";
        if (error.code === 'auth/user-not-found') message = "Pengguna tidak ditemukan.";
        if (error.code === 'auth/wrong-password') message = "Password salah.";
        if (error.code === 'auth/email-already-in-use') message = "Email sudah terdaftar.";
        if (error.code === 'auth/weak-password') message = "Password terlalu lemah (min. 6 karakter).";
        console.error("Gagal Autentikasi:", error);
        setAuthMessage(`Gagal: ${message}`);
    } finally {
        setIsAuthLoading(false);
    }
  };

  const handleLogout = useCallback(async () => {
    if (!auth) return;
    try {
        await signOut(auth);
        setIsLoggedIn(false);
        // Otomatis akan masuk anonim melalui onAuthStateChanged
        setAuthMessage("Anda telah logout. Silakan login kembali.");
        setActiveTab('dashboard'); // Arahkan ke dashboard/login screen
    } catch (error) {
        console.error("Gagal logout:", error);
        alert("Gagal melakukan logout.");
    }
  }, [auth]);

  // --- HANDLERS UTAMA ---
  
  // Menangani Penyimpanan Profil
  const handleSaveProfile = useCallback(async (e) => {
    e.preventDefault();
    if (!db || !userId) {
      setProfileMessage("Gagal: Koneksi Firebase belum siap.");
      return;
    }
    if (!userName.trim()) {
      setProfileMessage("Nama tidak boleh kosong.");
      return;
    }

    setIsProfileLoading(true);
    setProfileMessage('');

    try {
      const profileRef = getProfileDocRef();
      if (!profileRef) return;

      const profileData = {
        name: userName.trim(),
        phone: userPhone.trim(),
        updatedAt: serverTimestamp(),
      };

      await setDoc(profileRef, profileData, { merge: true });

      setCurrentName(userName.trim());
      setProfileMessage("Profil berhasil disimpan!");
    } catch (error) {
      console.error("Gagal menyimpan profil:", error);
      setProfileMessage(`Gagal menyimpan profil: ${error.message}`);
    } finally {
      setIsProfileLoading(false);
    }
  }, [db, userId, userName, userPhone, getProfileDocRef]);
  
  // Menangani Permintaan Penjemputan
  const handlePickupRequest = useCallback(async (e, isSpecial = false) => {
    e.preventDefault();
    if (!isLoggedIn) {
        setPickupUiMessage("Harap Login untuk mendaftar penjemputan.");
        return;
    }

    const location = isSpecial ? newLocation : e.target.location.value;
    const weight = isSpecial ? parseFloat(newWeight) : parseFloat(e.target.weight.value);
    const wasteType = isSpecial ? newWasteType : e.target.wasteType.value;

    if (!db || !userId) {
      setPickupUiMessage("Gagal: Koneksi Firebase belum siap.");
      return;
    }

    if (!location || weight <= 0 || wasteType === '') {
      setPickupUiMessage("Mohon lengkapi lokasi, jenis sampah, dan berat estimasi.");
      return;
    }

    isSpecial ? setIsPickupLoading(true) : setUiMessage('');
    const setMessage = isSpecial ? setPickupUiMessage : setUiMessage;
    setMessage('');

    try {
      const pickupData = {
        userId: userId,
        requesterName: currentName,
        location: location,
        weight: weight,
        wasteType: wasteType,
        status: 'Menunggu',
        timestamp: serverTimestamp(),
        notes: isSpecial ? "Permintaan penjemputan khusus (E-waste/Besar)." : "Permintaan penjemputan reguler.",
        isSpecialRequest: isSpecial
      };

      const pickupRef = collection(db, `artifacts/${appId}/users/${userId}/pickup_requests`);
      
      await addDoc(pickupRef, pickupData);
      
      setMessage(`Permintaan penjemputan ${isSpecial ? 'khusus' : 'reguler'} berhasil didaftarkan!`);
      if (isSpecial) {
          setNewLocation('');
          setNewWeight(0);
          setNewWasteType('E-Waste');
      }
    } catch (error) {
      console.error("Gagal mendaftar penjemputan:", error);
      setMessage(`Gagal mendaftar penjemputan: ${error.message}`);
    } finally {
      if (isSpecial) setIsPickupLoading(false);
    }
  }, [db, userId, currentName, newLocation, newWeight, newWasteType, isLoggedIn]);

  // Menangani Penukaran Poin
  const handleRedeem = useCallback(async (reward) => {
    if (!isLoggedIn) {
        alert("Harap Login untuk menukarkan poin.");
        return;
    }
    if (!db || !userId) {
        alert("Gagal: Koneksi Firebase belum siap.");
        return;
    }
    
    // Ganti dengan modal kustom di aplikasi nyata
    if (!window.confirm(`Yakin ingin menukarkan ${reward.points} Poin untuk ${reward.name}?`)) {
        return;
    }
    
    const profileRef = getProfileDocRef();
    if (!profileRef) return;

    try {
        // Menggunakan runTransaction untuk memastikan pengurangan poin yang aman (atomic operation)
        await runTransaction(db, async (transaction) => {
            const profileDoc = await transaction.get(profileRef);
            if (!profileDoc.exists()) throw new Error("Profil pengguna tidak ditemukan!");
            
            const currentPoints = profileDoc.data().totalPoints || 0;
            const newPoints = currentPoints - reward.points;
            
            if (newPoints < 0) throw new Error("Poin tidak mencukupi saat transaksi."); 
            
            transaction.update(profileRef, {
                totalPoints: newPoints,
                lastRedeem: serverTimestamp(),
            });
        });
        
        // Ganti dengan modal kustom di aplikasi nyata
        alert(`Berhasil menukarkan ${reward.points} Poin untuk ${reward.name}! Reward akan diproses.`);
    } catch (error) {
        console.error("Gagal menukarkan poin:", error);
        alert(`Gagal menukarkan poin: ${error.message}`);
    }
  }, [db, userId, getProfileDocRef, isLoggedIn]);

  // Menangani Pengiriman Review
  const handleSubmitReview = useCallback(async (e) => {
    e.preventDefault();
    if (!isLoggedIn) {
        setReviewMessage("Harap Login untuk memberikan review.");
        return;
    }
    if (!db || !userId) {
      setReviewMessage("Gagal: Koneksi Firebase belum siap.");
      return;
    }
    if (newRating === 0 || !newReviewText.trim()) {
        setReviewMessage("Rating bintang dan teks review tidak boleh kosong.");
        return;
    }
    
    setIsReviewLoading(true);
    setReviewMessage('');

    try {
        const reviewCollectionRef = getPublicReviewCollectionRef();
        if (!reviewCollectionRef) return;
        
        const reviewData = {
            userId: userId,
            userName: currentName,
            rating: newRating,
            reviewText: newReviewText.trim(),
            timestamp: serverTimestamp(),
        };

        await addDoc(reviewCollectionRef, reviewData);
        
        setReviewMessage("Review berhasil dikirim! Terima kasih atas masukan Anda.");
        setNewRating(0);
        setNewReviewText('');
    } catch (error) {
        console.error("Gagal mengirim review:", error);
        setReviewMessage(`Gagal mengirim review: ${error.message}`);
    } finally {
        setIsReviewLoading(false);
    }
  }, [db, userId, currentName, newRating, newReviewText, getPublicReviewCollectionRef, isLoggedIn]);


  // --- FORMATTING & UTILITIES ---
  
  const formatDate = (timestamp) => {
    if (!timestamp) return "N/A";
    const date = typeof timestamp.toDate === 'function' ? timestamp.toDate() : new Date();
    return date.toLocaleString('id-ID', {
      year: 'numeric', month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit'
    });
  };

  const formatRupiah = (number) => {
    // Format tanpa simbol mata uang, hanya angka dan pemisah ribuan
    return new Intl.NumberFormat('id-ID', { minimumFractionDigits: 0 }).format(number);
  }

  const getStatusColor = (status) => {
    switch (status) {
      case 'Selesai':
        return 'bg-green-100 text-green-800 border-green-300';
      case 'Menunggu':
        return 'bg-yellow-100 text-yellow-800 border-yellow-300';
      case 'Diambil':
        return 'bg-blue-100 text-blue-800 border-blue-300';
      default:
        return 'bg-gray-100 text-gray-800 border-gray-300';
    }
  };


  // --- SUB-COMPONENTS FOR TABS ---
  
  // Komponen Autentikasi (Login/Register)
  const AuthComponent = () => (
    <div className="bg-white p-6 rounded-xl shadow-2xl max-w-sm mx-auto border-t-4 border-blue-500">
        <h2 className="text-2xl font-bold mb-6 text-gray-800 text-center">
            {isRegisterMode ? 'Daftar Akun Baru' : 'Masuk ke EcoTrace'}
        </h2>
        <form onSubmit={handleAuth} className="space-y-4">
            <div>
                <label htmlFor="auth-email" className="block text-sm font-medium text-gray-700">Email</label>
                <input
                    type="email"
                    id="auth-email"
                    value={authEmail}
                    onChange={(e) => setAuthEmail(e.target.value)}
                    placeholder="nama@contoh.com"
                    required
                    className="mt-1 focus:ring-blue-500 focus:border-blue-500 block w-full sm:text-sm border-gray-300 rounded-md p-3"
                />
            </div>
            <div>
                <label htmlFor="auth-password" className="block text-sm font-medium text-gray-700">Password</label>
                <input
                    type="password"
                    id="auth-password"
                    value={authPassword}
                    onChange={(e) => setAuthPassword(e.target.value)}
                    placeholder="Minimal 6 karakter"
                    required
                    className="mt-1 focus:ring-blue-500 focus:border-blue-500 block w-full sm:text-sm border-gray-300 rounded-md p-3"
                />
            </div>
            <button
                type="submit"
                disabled={isAuthLoading || !isAuthReady || !auth}
                className={`w-full flex justify-center items-center py-3 px-4 border border-transparent rounded-lg shadow-md text-sm font-medium text-white transition duration-150 ease-in-out 
                  ${isAuthLoading ? 'bg-gray-400' : 'bg-blue-600 hover:bg-blue-700 focus:ring-blue-500'}`
                }
            >
                {isAuthLoading ? <Loader2 className="w-5 h-5 mr-2 animate-spin" /> : <Mail className="w-5 h-5 mr-2" />}
                {isRegisterMode ? 'Daftar' : 'Masuk'}
            </button>
        </form>

        {authMessage && (
          <div className={`mt-4 p-3 rounded-md text-sm text-center ${authMessage.includes('Gagal') ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'}`}>
            {authMessage}
          </div>
        )}
        
        <button
            onClick={() => { setIsRegisterMode(!isRegisterMode); setAuthMessage(''); }}
            className="w-full mt-4 text-sm text-blue-600 hover:text-blue-800 transition duration-150"
        >
            {isRegisterMode ? 'Sudah punya akun? Masuk' : 'Belum punya akun? Daftar Sekarang'}
        </button>
    </div>
  );

  // Komponen untuk Tab Dashboard
  const DashboardTab = useMemo(() => (
    <div className="space-y-8">
        {/* Form Manajemen Profil Pengguna */}
        <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-yellow-500">
            <h2 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
              <User className="w-5 h-5 mr-2 text-yellow-500" /> Kelola Profil
            </h2>
            {isLoggedIn ? (
                <form onSubmit={handleSaveProfile} className="space-y-3">
                    <div>
                      <label htmlFor="name" className="block text-sm font-medium text-gray-700">Nama Lengkap</label>
                      <input
                        type="text"
                        id="name"
                        value={userName}
                        onChange={(e) => setUserName(e.target.value)}
                        placeholder="Nama Anda"
                        required
                        className="focus:ring-yellow-500 focus:border-yellow-500 block w-full sm:text-sm border-gray-300 rounded-md p-2 mt-1"
                      />
                    </div>
                    <div>
                      <label htmlFor="phone" className="block text-sm font-medium text-gray-700">Nomor Telepon</label>
                      <input
                        type="tel"
                        id="phone"
                        value={userPhone}
                        onChange={(e) => setUserPhone(e.target.value)}
                        placeholder="Cth: 081234567890"
                        className="focus:ring-yellow-500 focus:border-yellow-500 block w-full sm:text-sm border-gray-300 rounded-md p-2 mt-1"
                      />
                    </div>
                    <button
                        type="submit"
                        disabled={isProfileLoading || !isAuthReady || !db}
                        className={`w-full flex justify-center items-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white transition duration-150 ease-in-out 
                          ${isProfileLoading ? 'bg-gray-400' : 'bg-yellow-600 hover:bg-yellow-700 focus:ring-yellow-500'}
                        `}
                    >
                        {isProfileLoading ? <Loader2 className="w-5 h-5 mr-2 animate-spin" /> : <Save className="w-5 h-5 mr-2" />} 
                        {isProfileLoading ? 'Menyimpan...' : 'Simpan Profil'}
                    </button>
                    {profileMessage && (
                      <div className={`mt-3 p-2 rounded-md text-xs ${profileMessage.includes('Gagal') ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'}`}>
                        {profileMessage}
                      </div>
                    )}
                </form>
            ) : (
                <p className="text-center text-red-500 font-medium">Harap Login untuk mengelola profil.</p>
            )}
        </div>
        
      {/* Bagian KPI / Ringkasan Bank Sampah */}
      <div className="grid grid-cols-1 sm:grid-cols-3 gap-4">
        <KPICard 
          title="Total Saldo Poin"
          value={formatRupiah(totalPoints)}
          unit="Poin yang terkumpul"
          color="bg-red-500"
          icon={DollarSign}
        />
        <KPICard 
          title="Total Berat Sampah"
          value={totalWeight.toFixed(1)}
          unit="Kg sampah yang telah disetorkan"
          color="bg-yellow-500"
          icon={Package}
        />
        <KPICard 
          title="Penjemputan Selesai"
          value={completedPickupsCount}
          unit="Permintaan yang sukses"
          color="bg-green-500"
          icon={CheckCircle}
        />
      </div>
      
      {/* Form Pendaftaran Penjemputan REGULER */}
      <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-green-500">
        <h2 className="text-xl font-semibold mb-6 text-gray-800 flex items-center">
          <Truck className="w-5 h-5 mr-2 text-green-500" /> Daftar Penjemputan Sampah Reguler
        </h2>
        <form onSubmit={(e) => handlePickupRequest(e, false)} className="space-y-4">
          <div>
            <label htmlFor="location" className="block text-sm font-medium text-gray-700">Lokasi Penjemputan</label>
            <input type="text" id="location" name="location" placeholder="Masukkan alamat lengkap" required className="mt-1 focus:ring-green-500 focus:border-green-500 block w-full sm:text-sm border-gray-300 rounded-md p-2"/>
          </div>
          
          <div className="grid grid-cols-2 gap-4">
            <div>
              <label htmlFor="wasteType" className="block text-sm font-medium text-gray-700">Jenis Sampah</label>
              <select id="wasteType" name="wasteType" className="mt-1 block w-full py-2 px-3 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-green-500 focus:border-green-500 sm:text-sm">
                <option>Plastik</option>
                <option>Kertas</option>
                <option>Kaca</option>
                <option>Logam</option>
              </select>
            </div>
            <div>
              <label htmlFor="weight" className="block text-sm font-medium text-gray-700">Berat Estimasi (kg)</label>
              <input type="number" id="weight" name="weight" min="0.1" step="0.1" required className="mt-1 focus:ring-green-500 focus:border-green-500 block w-full sm:text-sm border-gray-300 rounded-md p-2"/>
            </div>
          </div>

          <button
            type="submit"
            disabled={!isAuthReady || !db || !isLoggedIn}
            className={`w-full flex justify-center items-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white transition duration-150 ease-in-out 
              ${!isAuthReady || !db || !isLoggedIn ? 'bg-gray-400 cursor-not-allowed' : 'bg-green-600 hover:bg-green-700 focus:ring-green-500'}`
            }
          >
            <Recycle className="w-5 h-5 mr-2" /> Daftar Penjemputan
          </button>
        </form>
        {uiMessage && (
          <div className={`mt-4 p-3 rounded-md text-sm ${uiMessage.includes('Gagal') ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'}`}>
            {uiMessage}
          </div>
        )}
      </div>

      {/* Riwayat Penjemputan */}
      <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-blue-500">
        <h2 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
          <Trash2 className="w-5 h-5 mr-2 text-blue-500" /> Riwayat Permintaan Penjemputan ({pickups.length})
        </h2>
        {!isLoggedIn ? (
            <p className="text-red-500 text-center py-4">Harap Login untuk melihat riwayat.</p>
        ) : pickups.length === 0 && isAuthReady ? (
          <p className="text-gray-500 text-center py-4">Belum ada riwayat penjemputan yang terdaftar.</p>
        ) : !isAuthReady ? (
          <p className="text-gray-500 text-center py-4">Memuat data...</p>
        ) : (
          <div className="space-y-3 max-h-96 overflow-y-auto pr-2">
            {pickups.map((pickup) => (
              <div key={pickup.id} className={`p-3 border rounded-lg shadow-sm flex justify-between items-center ${getStatusColor(pickup.status)}`}>
                <div className="flex flex-col">
                  <p className="font-semibold text-gray-700">{pickup.location} - <span className="text-sm font-normal text-gray-500">{pickup.wasteType}</span></p>
                  <div className="text-xs text-gray-500 flex items-center mt-1">
                    <Clock className="w-3 h-3 mr-1" /> {formatDate(pickup.timestamp)}
                  </div>
                </div>
                <div className="text-right">
                    <p className="font-bold text-lg text-blue-600">{pickup.weight.toFixed(1)} kg</p>
                    <span className={`px-2 py-0.5 text-xs font-medium rounded-full ${getStatusColor(pickup.status)}`}>
                        {pickup.status}
                    </span>
                </div>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  ), [totalPoints, totalWeight, completedPickupsCount, uiMessage, pickups, isAuthReady, handlePickupRequest, isLoggedIn, userName, userPhone, isProfileLoading, profileMessage, handleSaveProfile]);

  // Komponen untuk Tab Pelacakan Truk (Simulasi)
  const TrackingTab = () => {
    const [timeToArrival, setTimeToArrival] = useState(15); // Waktu dalam menit
    const [status, setStatus] = useState('Dalam Perjalanan');

    useEffect(() => {
        // Reset state saat mount/tab aktif
        setTimeToArrival(15); 
        setStatus('Dalam Perjalanan');
        
        const interval = setInterval(() => {
            setTimeToArrival(prevTime => {
                const newTime = prevTime > 0 ? prevTime - 1 : 0;
                if (newTime === 5) {
                    setStatus('Hampir Tiba (5 Menit)');
                } else if (newTime === 0) {
                    setStatus('Telah Tiba di Lokasi');
                    clearInterval(interval);
                }
                return newTime;
            });
        }, 3000); // Update setiap 3 detik (simulasi 1 menit berlalu)

        return () => clearInterval(interval);
    }, []); 

    if (!isLoggedIn) {
        return (
            <div className="max-w-2xl mx-auto bg-red-100 p-6 rounded-xl shadow-lg border-l-4 border-red-500 text-center">
                <p className="text-red-800 font-bold text-lg">Akses Ditolak.</p>
                <p className="text-red-700 mt-2">Harap Login untuk melacak penjemputan Anda.</p>
            </div>
        );
    }

    return (
      <div className="max-w-2xl mx-auto bg-white p-6 rounded-xl shadow-lg border-t-4 border-orange-500">
        <h2 className="text-2xl font-semibold mb-6 text-gray-800 flex items-center">
          <Truck className="w-6 h-6 mr-2 text-orange-500" /> Pelacakan Truk Real-time
        </h2>
        
        <div className="text-center p-6 border border-gray-200 rounded-lg bg-orange-50">
            <Truck className={`w-12 h-12 mx-auto mb-4 ${status === 'Telah Tiba di Lokasi' ? 'text-green-600' : 'text-orange-500 animate-bounce'}`} />
            <p className="text-sm text-gray-600 font-medium">Perkiraan Waktu Tiba (ETA)</p>
            <div className="text-4xl font-extrabold mt-2 mb-4">
                {timeToArrival > 0 ? `${timeToArrival} menit` : 'Telah Tiba!'}
            </div>
            <span className={`px-3 py-1 font-semibold rounded-full text-white ${
                timeToArrival > 5 ? 'bg-orange-500' : 
                timeToArrival > 0 ? 'bg-yellow-500' : 'bg-green-600'
            }`}>
                {status}
            </span>
        </div>
        
        <div className="mt-6 text-sm text-gray-600 space-y-2">
            <p className="flex items-center">
                <Clock3 className="w-4 h-4 mr-2 text-gray-500" /> **Waktu Pembaruan:** Setiap 3 detik (Simulasi)
            </p>
            <p className="flex items-center">
                <MapPin className="w-4 h-4 mr-2 text-gray-500" /> **Tujuan:** Lokasi penjemputan terdekat di rute Anda.
            </p>
            <p className="p-3 bg-gray-100 rounded-md italic">
                Anda dapat melacak pergerakan truk di peta di sini (Simulasi peta tidak diimplementasikan).
            </p>
            
        </div>
      </div>
    );
  };
  
  // Komponen untuk Tab Ekosistem & Reward (termasuk Bank Sampah, Reward, Jemput Khusus)
  const EcosystemTab = () => {
    const [subTab, setSubTab] = useState('bank'); // 'bank', 'reward', 'pickup', 'education'

    const renderSubContent = () => {
      if (!isLoggedIn) {
        return (
            <div className="text-center p-8 bg-red-50 rounded-lg">
                <p className="text-red-700 font-bold">Harap Login untuk mengakses Ekosistem & Reward.</p>
                <p className="text-sm text-red-600 mt-1">Anda dapat mendaftar untuk mendapatkan Poin dan Reward!</p>
            </div>
        );
      }

      switch (subTab) {
        case 'bank':
          return (
            <div className="space-y-4">
                <h3 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
                    <MapPin className="w-5 h-5 mr-2 text-blue-500" /> Informasi Lokasi Bank Sampah
                </h3>
                <p className="text-sm text-gray-600 mb-4">Temukan Bank Sampah dan TPS 3R terdekat (Data Mock).</p>
                {MOCK_BANK_SAMPAH.map((bank, index) => (
                    <div key={index} className="p-4 border rounded-lg hover:bg-gray-50 flex justify-between items-center transition duration-150">
                        <div>
                            <p className="font-semibold text-green-600">{bank.name}</p>
                            <p className="text-xs text-gray-500 mt-1">{bank.address}</p>
                        </div>
                        <span className="font-bold text-sm text-blue-500">{bank.distance}</span>
                    </div>
                ))}
            </div>
          );
        case 'reward':
            return (
                <div className="space-y-4">
                    <h3 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
                        <Gift className="w-5 h-5 mr-2 text-red-500" /> Tukar Poin Anda
                    </h3>
                    <p className="text-sm text-gray-600 mb-4">Saldo Poin Anda Saat Ini: <span className="font-bold text-red-600">{formatRupiah(totalPoints)} Poin</span></p>
                    
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        {MOCK_REWARDS.map((reward) => (
                            <div key={reward.id} className="p-4 border rounded-xl shadow-md bg-white flex flex-col justify-between">
                                <div>
                                    <p className="text-lg font-semibold text-gray-800">{reward.name}</p>
                                    <p className="text-xs text-gray-500 mt-1">{reward.description}</p>
                                </div>
                                <div className="mt-3 flex justify-between items-center">
                                    <span className="text-sm font-bold text-red-500">{formatRupiah(reward.points)} Poin</span>
                                    <button
                                        onClick={() => handleRedeem(reward)}
                                        disabled={totalPoints < reward.points}
                                        className={`py-1 px-3 rounded-full text-white text-sm font-medium transition duration-150 
                                            ${totalPoints < reward.points ? 'bg-gray-400 cursor-not-allowed' : 'bg-red-500 hover:bg-red-600'}`
                                        }
                                    >
                                        Tukar
                                    </button>
                                </div>
                            </div>
                        ))}
                    </div>
                </div>
            );
        case 'pickup':
            return (
                <div className="space-y-4">
                    <h3 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
                        <Truck className="w-5 h-5 mr-2 text-purple-500" /> Pendaftaran Penjemputan Khusus
                    </h3>
                    <p className="text-sm text-gray-600 mb-4">Untuk sampah berukuran besar atau khusus (misalnya: elektronik, perabotan, jual beli sampah terpilah dalam jumlah besar).</p>
                    
                    <form onSubmit={(e) => handlePickupRequest(e, true)} className="space-y-4 p-4 bg-purple-50 rounded-lg">
                        <div>
                            <label htmlFor="newLocation" className="block text-sm font-medium text-gray-700">Lokasi Penjemputan</label>
                            <input
                                type="text"
                                id="newLocation"
                                value={newLocation}
                                onChange={(e) => setNewLocation(e.target.value)}
                                placeholder="Alamat lengkap penjemputan"
                                required
                                className="mt-1 focus:ring-purple-500 focus:border-purple-500 block w-full sm:text-sm border-gray-300 rounded-md p-2"
                            />
                        </div>
                        <div className="grid grid-cols-2 gap-4">
                            <div>
                                <label htmlFor="newWasteType" className="block text-sm font-medium text-gray-700">Jenis Sampah</label>
                                <select
                                    id="newWasteType"
                                    value={newWasteType}
                                    onChange={(e) => setNewWasteType(e.target.value)}
                                    className="mt-1 block w-full py-2 px-3 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-purple-500 focus:border-purple-500 sm:text-sm"
                                >
                                    <option>E-Waste</option>
                                    <option>Perabotan</option>
                                    <option>Besi Tua</option>
                                    <option>Plastik (Jumlah Besar)</option>
                                </select>
                            </div>
                            <div>
                                <label htmlFor="newWeight" className="block text-sm font-medium text-gray-700">Berat Estimasi (kg)</label>
                                <input
                                    type="number"
                                    id="newWeight"
                                    value={newWeight}
                                    onChange={(e) => setNewWeight(e.target.value)}
                                    min="0.1"
                                    step="0.1"
                                    required
                                    className="mt-1 focus:ring-purple-500 focus:border-purple-500 block w-full sm:text-sm border-gray-300 rounded-md p-2"
                                />
                            </div>
                        </div>

                        <button
                            type="submit"
                            disabled={isPickupLoading || !isAuthReady || !db || !isLoggedIn}
                            className={`w-full flex justify-center items-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white transition duration-150 ease-in-out 
                                ${isPickupLoading || !isLoggedIn ? 'bg-gray-400' : 'bg-purple-600 hover:bg-purple-700 focus:ring-purple-500'}`
                            }
                        >
                            {isPickupLoading ? <Loader2 className="w-5 h-5 mr-2 animate-spin" /> : <Recycle className="w-5 h-5 mr-2" />}
                            {isPickupLoading ? 'Mendaftar...' : 'Daftar Penjemputan Khusus'}
                        </button>
                        {pickupUiMessage && (
                            <div className={`mt-3 p-2 rounded-md text-xs ${pickupUiMessage.includes('Gagal') ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'}`}>
                                {pickupUiMessage}
                            </div>
                        )}
                    </form>
                </div>
            );
        case 'education':
            return (
                <div className="space-y-4">
                    <h3 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
                        <BookOpen className="w-5 h-5 mr-2 text-teal-500" /> Informasi dan Edukasi
                    </h3>
                    <p className="text-sm text-gray-600 mb-4">Tingkatkan pengetahuan Anda tentang daur ulang dan pengelolaan sampah (Fitur Edukasi).</p>
                    
                    {MOCK_ARTICLES.map((article, index) => (
                        <div key={index} className="p-4 border rounded-lg hover:bg-teal-50 flex justify-between items-center transition duration-150 border-l-4 border-teal-400">
                            <div>
                                <p className="font-semibold text-gray-800">{article.title}</p>
                                <p className="text-xs text-gray-500 mt-1">Sumber: {article.source} | {article.date}</p>
                            </div>
                            <BookOpen className="w-5 h-5 text-teal-400" />
                        </div>
                    ))}
                </div>
            );
        default:
            return null;
      }
    };

    const subTabs = [
        { id: 'bank', name: 'Bank Sampah', icon: MapPin },
        { id: 'reward', name: 'Reward Poin', icon: Gift },
        { id: 'pickup', name: 'Jemput Khusus/Jual Beli', icon: Truck },
        { id: 'education', name: 'Edukasi', icon: BookOpen },
    ];

    return (
      <div className="max-w-4xl mx-auto bg-white p-6 rounded-xl shadow-lg border-t-4 border-emerald-500">
        <h2 className="text-2xl font-bold mb-4 text-gray-800">Ekosistem & Layanan EcoTrace</h2>

        {/* Sub-Tab Navigation */}
        <div className="flex border-b border-gray-200 mb-6 overflow-x-auto">
          {subTabs.map((tab) => (
            <button
              key={tab.id}
              onClick={() => setSubTab(tab.id)}
              className={`flex-shrink-0 flex items-center px-4 py-2 text-sm font-medium transition duration-150 ${
                subTab === tab.id
                  ? 'border-b-2 border-emerald-500 text-emerald-600'
                  : 'text-gray-500 hover:text-gray-700'
              }`}
            >
              <tab.icon className="w-4 h-4 mr-2" />
              {tab.name}
            </button>
          ))}
        </div>

        {renderSubContent()}
      </div>
    );
  };
  
  // Komponen untuk Tab Review & Rating
  const ReviewTab = () => {
    const averageRating = useMemo(() => {
        if (allReviews.length === 0) return 0;
        const total = allReviews.reduce((sum, review) => sum + (review.rating || 0), 0);
        return (total / allReviews.length).toFixed(1);
    }, [allReviews]);

    return (
      <div className="max-w-4xl mx-auto space-y-8">
        {/* Ringkasan Rating */}
        <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-fuchsia-500 text-center">
            <h2 className="text-2xl font-bold text-gray-800">Ulasan Pengguna EcoTrace</h2>
            <div className="mt-4 flex flex-col items-center">
                <p className="text-5xl font-extrabold text-fuchsia-600">{averageRating}</p>
                <StarRating rating={Math.round(averageRating)} size={8} />
                <p className="text-sm text-gray-500 mt-2">Dari {allReviews.length} Ulasan</p>
            </div>
        </div>

        {/* Form Input Review */}
        <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-indigo-500">
            <h3 className="text-xl font-semibold mb-4 text-gray-800 flex items-center">
                <Star className="w-5 h-5 mr-2 text-indigo-500" /> Beri Ulasan Anda
            </h3>
            {isLoggedIn ? (
                <form onSubmit={handleSubmitReview} className="space-y-4">
                    <div>
                        <label className="block text-sm font-medium text-gray-700 mb-1">Rating Bintang</label>
                        <div className="flex space-x-1">
                            {[...Array(5)].map((_, i) => (
                                <Star
                                    key={i}
                                    onClick={() => setNewRating(i + 1)}
                                    className={`w-8 h-8 cursor-pointer transition duration-150 ${
                                        i < newRating ? 'text-yellow-400 fill-yellow-400' : 'text-gray-300 hover:text-yellow-300 hover:fill-yellow-300'
                                    }`}
                                />
                            ))}
                        </div>
                    </div>
                    <div>
                        <label htmlFor="reviewText" className="block text-sm font-medium text-gray-700">Komentar/Review</label>
                        <textarea
                            id="reviewText"
                            value={newReviewText}
                            onChange={(e) => setNewReviewText(e.target.value)}
                            rows="3"
                            placeholder="Bagaimana pengalaman Anda menggunakan EcoTrace?"
                            required
                            className="mt-1 focus:ring-indigo-500 focus:border-indigo-500 block w-full sm:text-sm border-gray-300 rounded-md p-2"
                        ></textarea>
                    </div>
                    <button
                        type="submit"
                        disabled={isReviewLoading || !isAuthReady || newRating === 0 || !newReviewText.trim()}
                        className={`w-full flex justify-center items-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white transition duration-150 ease-in-out 
                            ${isReviewLoading ? 'bg-gray-400' : 'bg-indigo-600 hover:bg-indigo-700 focus:ring-indigo-500'}`
                        }
                    >
                        {isReviewLoading ? <Loader2 className="w-5 h-5 mr-2 animate-spin" /> : <Star className="w-5 h-5 mr-2" />}
                        {isReviewLoading ? 'Mengirim...' : 'Kirim Ulasan'}
                    </button>
                    {reviewMessage && (
                      <div className={`mt-3 p-2 rounded-md text-xs ${reviewMessage.includes('Gagal') ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'}`}>
                        {reviewMessage}
                      </div>
                    )}
                </form>
            ) : (
                <p className="text-center text-red-500 font-medium">Harap Login untuk memberikan ulasan.</p>
            )}
        </div>
        
        {/* Daftar Review */}
        <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-orange-500">
            <h3 className="text-xl font-semibold mb-4 text-gray-800">Daftar Semua Ulasan</h3>
            <div className="space-y-4 max-h-96 overflow-y-auto pr-2">
                {allReviews.length === 0 ? (
                    <p className="text-gray-500 text-center py-4">Belum ada ulasan yang tersedia.</p>
                ) : (
                    allReviews.map((review) => (
                        <div key={review.id} className="p-4 border rounded-lg bg-gray-50">
                            <div className="flex justify-between items-center mb-2">
                                <div className="font-semibold text-gray-800 flex items-center">
                                    <User className="w-4 h-4 mr-2 text-blue-500" /> {review.userName || 'Pengguna'}
                                </div>
                                <StarRating rating={review.rating} size={4} />
                            </div>
                            <p className="text-sm text-gray-600 italic">"{review.reviewText}"</p>
                            <p className="text-xs text-gray-400 mt-2 text-right">{formatDate(review.timestamp)}</p>
                        </div>
                    ))
                )}
            </div>
        </div>
      </div>
    );
  };


  // --- RENDER UTAMA ---
  const renderContent = () => {
    if (!isAuthReady) {
      return (
        <div className="text-center py-10">
          <Loader2 className="w-10 h-10 mx-auto text-green-500 animate-spin" />
          <p className="mt-4 text-lg text-gray-600">Memuat data dan mengautentikasi pengguna...</p>
        </div>
      );
    }
    
    // Tampilkan Komponen Login/Register jika pengguna Anonim
    if (!isLoggedIn && !isAnonymous && activeTab === 'dashboard') {
        return <AuthComponent />;
    }

    switch (activeTab) {
      case 'dashboard':
        return <div className="max-w-4xl mx-auto">{DashboardTab}</div>;
      case 'tracking':
        return <TrackingTab />;
      case 'ecosystem':
        return <EcosystemTab />;
      case 'review':
        return <ReviewTab />;
      default:
        return <div className="max-w-4xl mx-auto">{DashboardTab}</div>;
    }
  };

  const navItems = [
    { id: 'dashboard', name: 'Dashboard', icon: Recycle },
    { id: 'tracking', name: 'Pelacakan', icon: Truck },
    { id: 'ecosystem', name: 'Ekosistem & Reward', icon: Gift },
    { id: 'review', name: 'Review', icon: Star },
  ];


  return (
    <div className="min-h-screen bg-gray-50 p-4 font-sans">
      <script src="https://cdn.tailwindcss.com"></script>
      
      <header className="text-center mb-6 max-w-5xl mx-auto">
        <div className="flex items-center justify-center">
            <h1 className="text-4xl font-extrabold text-green-600 flex items-center">
              <Recycle className="w-8 h-8 mr-2" /> EcoTrace Super Dashboard
            </h1>
            {isLoggedIn && (
                <button
                    onClick={handleLogout}
                    className="ml-4 p-2 bg-red-500 hover:bg-red-600 text-white rounded-lg text-sm font-medium flex items-center shadow-md transition duration-150"
                >
                    <LogOut className="w-4 h-4 mr-1" /> Logout
                </button>
            )}
        </div>
        <p className="text-lg text-gray-600 mt-2">Sistem Manajemen Sampah Cerdas Terpadu</p>
        <div className="mt-4 p-2 bg-green-50 text-green-700 rounded-lg shadow-inner text-sm">
          <User className="w-4 h-4 inline mr-1"/> <span className="font-bold">{currentName}</span> ({isLoggedIn ? 'Terdaftar' : 'Anonim'}) | (ID: <span className="font-mono text-xs break-all">{userId || "Memuat..."}</span>)
        </div>
      </header>

      {/* Navigasi Utama Tab */}
      <nav className="max-w-5xl mx-auto mb-8 bg-white p-2 rounded-xl shadow-lg flex justify-around overflow-x-auto">
        {navItems.map(item => (
          <button
            key={item.id}
            onClick={() => setActiveTab(item.id)}
            className={`flex-shrink-0 flex items-center justify-center p-3 rounded-lg text-sm font-medium transition duration-200 w-1/4
              ${activeTab === item.id 
                ? 'bg-green-100 text-green-700 shadow-md scale-105' 
                : 'text-gray-500 hover:bg-gray-100'
              }`
            }
          >
            <item.icon className="w-5 h-5 mr-2" />
            <span className="hidden sm:inline">{item.name}</span>
            <span className="sm:hidden">{item.name.split('&')[0].trim()}</span>
          </button>
        ))}
      </nav>

      {/* Konten Tab Aktif */}
      <div className="max-w-5xl mx-auto">
      
      {renderContent()}

      </div>
    </div>
  );
}

export default App;
