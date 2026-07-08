# ربط AI Topper Smart بقاعدة بيانات Firebase + تسجيل دخول جوجل

هذا الملف فيه كل الكود الجاهز. تحتاج فقط تسوي مشروع Firebase مجاني (5 دقائق، بحساب جوجل خاص فيك — ما أقدر أسويه بدالك لأنه يحتاج تفعيل من حسابك).

## 1) إنشاء المشروع (تسويها أنت مرة وحدة)

1. روح **console.firebase.google.com**
2. **Add project** → اختر اسم (مثلاً `ai-topper-smart`) → أكمل الخطوات
3. من القائمة الجانبية: **Build → Authentication** → **Get started** → فعّل **Google** كطريقة دخول
4. من القائمة: **Build → Firestore Database** → **Create database** → اختر وضع **Production**
5. ارجع لصفحة المشروع الرئيسية → أيقونة **</>** (Web) → سجّل تطبيق جديد → انسخ كائن `firebaseConfig` اللي يطلع لك (يشبه الكود تحت)

## 2) الكود الجاهز — ألصقه قبل `</body>` في ملف الموقع

```html
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
  import {
    getAuth, GoogleAuthProvider, signInWithPopup, onAuthStateChanged, signOut
  } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
  import {
    getFirestore, doc, setDoc, getDoc, updateDoc
  } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

  // 👇 الصق هنا القيم اللي نسختها من Firebase Console (وليست سرية، آمنة للنشر العام)
  const firebaseConfig = {
    apiKey: "PASTE_HERE",
    authDomain: "PASTE_HERE",
    projectId: "PASTE_HERE",
    storageBucket: "PASTE_HERE",
    messagingSenderId: "PASTE_HERE",
    appId: "PASTE_HERE",
  };

  const app = initializeApp(firebaseConfig);
  const auth = getAuth(app);
  const db = getFirestore(app);
  const provider = new GoogleAuthProvider();

  // تسجيل الدخول بجوجل
  window.loginWithGoogle = async () => {
    const result = await signInWithPopup(auth, provider);
    await ensureUserDoc(result.user);
  };

  window.logout = () => signOut(auth);

  // ينشئ سجل بيانات خاص بالطالب أول مرة يسجل فيها دخول
  async function ensureUserDoc(user) {
    const ref = doc(db, "students", user.uid);
    const snap = await getDoc(ref);
    if (!snap.exists()) {
      await setDoc(ref, {
        name: user.displayName,
        email: user.email,
        joinedAt: new Date().toISOString(),
        streak: 0,
        points: 0,
        grades: {},
      });
    }
  }

  // مثال: تحديث نقاط الطالب بعد اختبار
  window.saveExamResult = async (uid, examName, score) => {
    const ref = doc(db, "students", uid);
    await updateDoc(ref, { [`grades.${examName}`]: score });
  };

  // يتابع حالة الدخول تلقائياً ويحدّث الواجهة
  onAuthStateChanged(auth, (user) => {
    const el = document.getElementById("authStatus");
    if (el) el.textContent = user ? `مرحباً ${user.displayName} 👋` : "لم تسجّل الدخول";
  });
</script>
```

## 3) زر تسجيل الدخول (ألصقه بأي مكان بالواجهة)

```html
<button onclick="loginWithGoogle()">تسجيل الدخول بحساب جوجل</button>
<span id="authStatus"></span>
```

## ملاحظات أمان مهمة

- `firebaseConfig` (الموجود فوق) **آمن للنشر العام** ولا يشبه مفاتيح Groq/Anthropic — هذا مصمم من جوجل ليكون عاماً، لأن الحماية الفعلية تتم عبر "Firestore Security Rules" (تحدد مين يقدر يقرأ/يكتب بيانات مين)، مو عبر إخفاء الكود.
- **لازم** تضبط Security Rules في Firestore بعد الإعداد، وإلا أي شخص يقدر يقرأ بيانات الجميع. مثال بسيط وآمن:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /students/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

هذا يضمن كل طالب يشوف بياناته هو بس.

## الخطة القادمة

بمجرد ما يصير عندك firebaseConfig، عطني إياه (آمن، تقدر ترسله بدون قلق) وأدمجه مباشرة داخل `ai-topper-smart.html` وأربط كل الصفحات (النقاط، الشارات، نتائج الاختبارات، الملف الشخصي) تُقرأ وتُكتب من Firestore بدل البيانات الوهمية الحالية.
