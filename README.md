<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>GharKhata — Household Finance Portal</title>
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;600;700&family=DM+Sans:wght@300;400;500;600&family=DM+Mono:wght@400;500&display=swap" rel="stylesheet">

<!-- Firebase SDK -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
  import { getAuth, signInWithEmailAndPassword, signOut, onAuthStateChanged,
           sendPasswordResetEmail, createUserWithEmailAndPassword }
    from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
  import { getFirestore, collection, doc, setDoc, getDoc, addDoc, updateDoc,
           deleteDoc, onSnapshot, query, orderBy, serverTimestamp, getDocs }
    from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

  // ══════════════════════════════════════════════════
  // ✅ FIREBASE CONFIG — Connected to gharkhata-72a0c
  // ══════════════════════════════════════════════════
  const firebaseConfig = {
    apiKey: "AIzaSyDr9AgTrCbErS1kI7Szkj5I7ATl1VsjEGg",
    authDomain: "gharkhata-72a0c.firebaseapp.com",
    projectId: "gharkhata-72a0c",
    storageBucket: "gharkhata-72a0c.firebasestorage.app",
    messagingSenderId: "392824152428",
    appId: "1:392824152428:web:1d83f813794b923f649349",
    measurementId: "G-0QQGRCBWMX"
  };

  const app = initializeApp(firebaseConfig);
  const auth = getAuth(app);
  const db = getFirestore(app);

  // ══ GLOBAL STATE ══
  window.GK = {
    auth, db, user: null, role: null,
    transactions: [], accounts: [], budgets: [], loans: [],
    unsubscribers: [],
    activeAccTab: null
  };

  // ══════════════════════════════════════════════════
  // AUTH LISTENER
  // ══════════════════════════════════════════════════
  onAuthStateChanged(auth, async (user) => {
    if (user) {
      window.GK.user = user;
      // Fetch role from Firestore
      try {
        const snap = await getDoc(doc(db, 'users', user.uid));
        if (snap.exists()) {
          window.GK.role = snap.data().role || 'viewer';
          window.GK.displayName = snap.data().name || user.email;
        } else {
          // First-time admin setup: if no users doc at all, make this user admin
          const allSnap = await getDocs(collection(db, 'users'));
          if (allSnap.empty) {
            await setDoc(doc(db, 'users', user.uid), { role:'admin', name:'Admin', email: user.email, createdAt: serverTimestamp() });
            window.GK.role = 'admin';
            window.GK.displayName = 'Admin';
          } else {
            window.GK.role = 'viewer';
            window.GK.displayName = user.email;
          }
        }
      } catch(e) { window.GK.role = 'viewer'; }
      showApp();
      startListeners();
    } else {
      window.GK.user = null; window.GK.role = null;
      stopListeners();
      showAuth();
    }
  });

  // ══════════════════════════════════════════════════
  // REALTIME LISTENERS
  // ══════════════════════════════════════════════════
  function startListeners() {
    stopListeners();
    const add = (unsub) => window.GK.unsubscribers.push(unsub);

    add(onSnapshot(query(collection(db,'transactions'), orderBy('date','desc')), snap => {
      window.GK.transactions = snap.docs.map(d=>({id:d.id,...d.data()}));
      refreshCurrentPage();
    }));
    add(onSnapshot(collection(db,'accounts'), snap => {
      window.GK.accounts = snap.docs.map(d=>({id:d.id,...d.data()}));
      refreshCurrentPage();
    }));
    add(onSnapshot(collection(db,'budgets'), snap => {
      window.GK.budgets = snap.docs.map(d=>({id:d.id,...d.data()}));
      refreshCurrentPage();
    }));
    add(onSnapshot(collection(db,'loans'), snap => {
      window.GK.loans = snap.docs.map(d=>({id:d.id,...d.data()}));
      refreshCurrentPage();
    }));
    add(onSnapshot(collection(db,'users'), snap => {
      window.GK.allUsers = snap.docs.map(d=>({id:d.id,...d.data()}));
      if(document.getElementById('page-users')?.classList.contains('active')) renderUsers();
    }));
  }
  function stopListeners() {
    window.GK.unsubscribers.forEach(u=>u());
    window.GK.unsubscribers = [];
  }

  // ══════════════════════════════════════════════════
  // AUTH OPERATIONS
  // ══════════════════════════════════════════════════
  window.doLogin = async () => {
    const email = document.getElementById('login-email').value.trim();
    const pass  = document.getElementById('login-pass').value;
    const btn   = document.getElementById('login-btn');
    if(!email||!pass) return setAuthError('Enter email and password');
    btn.disabled=true; btn.textContent='Signing in…';
    try {
      await signInWithEmailAndPassword(auth, email, pass);
    } catch(e) {
      setAuthError(friendlyError(e.code));
      btn.disabled=false; btn.textContent='Sign In';
    }
  };

  window.doLogout = async () => {
    stopListeners();
    await signOut(auth);
  };

  window.doResetPassword = async () => {
    const email = document.getElementById('reset-email').value.trim();
    if(!email) return setAuthError('Enter your email address');
    try {
      await sendPasswordResetEmail(auth, email);
      document.getElementById('reset-success').style.display='block';
      document.getElementById('reset-success').textContent='✓ Password reset email sent! Check your inbox.';
    } catch(e) { setAuthError(friendlyError(e.code)); }
  };

  // Admin: create new user
  window.doCreateUser = async () => {
    if(window.GK.role!=='admin') return;
    const email = document.getElementById('new-user-email').value.trim();
    const pass  = document.getElementById('new-user-pass').value;
    const name  = document.getElementById('new-user-name').value.trim();
    const role  = document.getElementById('new-user-role').value;
    if(!email||!pass||!name) return showToast('⚠️ Fill all fields','error');

    // We use a secondary auth instance trick: create user then immediately sign back in
    const currentUser = auth.currentUser;
    const currentEmail = currentUser.email;
    const currentPassEl = document.getElementById('admin-reauth-pass');
    const currentPass = currentPassEl ? currentPassEl.value : '';

    try {
      // Create via REST (avoids signing out current user)
      const res = await fetch(`https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=${firebaseConfig.apiKey}`, {
        method:'POST', headers:{'Content-Type':'application/json'},
        body: JSON.stringify({ email, password: pass, returnSecureToken:true })
      });
      const data = await res.json();
      if(data.error) throw new Error(data.error.message);
      const uid = data.localId;
      // Save user profile in Firestore
      await setDoc(doc(db,'users',uid), { email, name, role, createdAt: serverTimestamp(), createdBy: currentUser.uid });
      closeModal('addUserModal');
      showToast(`✓ User "${name}" created as ${role}`, 'success');
    } catch(e) { showToast('Error: '+e.message, 'error'); }
  };

  window.doUpdateUserRole = async (uid, newRole) => {
    if(window.GK.role!=='admin') return;
    await updateDoc(doc(db,'users',uid), { role: newRole });
    showToast('✓ Role updated','success');
  };

  window.doDeleteUser = async (uid) => {
    if(window.GK.role!=='admin') return;
    if(uid===window.GK.user.uid) return showToast('Cannot delete yourself','error');
    await deleteDoc(doc(db,'users',uid));
    showToast('User removed from portal (auth account still exists)','info');
  };

  // ══════════════════════════════════════════════════
  // FIRESTORE CRUD — guarded by role
  // ══════════════════════════════════════════════════
  function isAdmin() { return window.GK.role === 'admin'; }
  function guardAdmin() { if(!isAdmin()){ showToast('👁️ View-only access. Ask admin to make changes.','error'); return false; } return true; }

  window.addTransaction = async (data) => {
    if(!guardAdmin()) return false;
    await addDoc(collection(db,'transactions'), { ...data, createdBy: window.GK.user.uid, createdAt: serverTimestamp() });
    return true;
  };
  window.updateTransaction = async (id, data) => {
    if(!guardAdmin()) return false;
    await updateDoc(doc(db,'transactions',id), { ...data, updatedBy: window.GK.user.uid, updatedAt: serverTimestamp() });
    return true;
  };
  window.deleteTransaction = async (id) => {
    if(!guardAdmin()) return false;
    await deleteDoc(doc(db,'transactions',id));
    return true;
  };

  window.addAccount = async (data) => {
    if(!guardAdmin()) return false;
    const ref = doc(collection(db,'accounts'));
    await setDoc(ref, { ...data, createdAt: serverTimestamp() });
    return true;
  };
  window.updateAccount = async (id, data) => {
    if(!guardAdmin()) return false;
    await updateDoc(doc(db,'accounts',id), data);
    return true;
  };
  window.deleteAccount = async (id) => {
    if(!guardAdmin()) return false;
    await deleteDoc(doc(db,'accounts',id));
    return true;
  };

  window.saveBudgetDoc = async (id, data) => {
    if(!guardAdmin()) return false;
    if(id) await updateDoc(doc(db,'budgets',id), data);
    else await addDoc(collection(db,'budgets'), { ...data, createdAt: serverTimestamp() });
    return true;
  };
  window.deleteBudgetDoc = async (id) => {
    if(!guardAdmin()) return false;
    await deleteDoc(doc(db,'budgets',id));
    return true;
  };

  window.saveLoanDoc = async (data) => {
    if(!guardAdmin()) return false;
    await addDoc(collection(db,'loans'), { ...data, createdAt: serverTimestamp() });
    return true;
  };
  window.deleteLoanDoc = async (id) => {
    if(!guardAdmin()) return false;
    await deleteDoc(doc(db,'loans',id));
    return true;
  };

  // ══ HELPERS ══
  function friendlyError(code) {
    const map = {
      'auth/user-not-found':'No account found with this email.',
      'auth/wrong-password':'Incorrect password.',
      'auth/invalid-email':'Invalid email address.',
      'auth/too-many-requests':'Too many attempts. Try again later.',
      'auth/invalid-credential':'Invalid email or password.',
      'auth/email-already-in-use':'This email is already registered.',
      'auth/weak-password':'Password must be at least 6 characters.',
      'auth/network-request-failed':'Network error. Check your connection.'
    };
    return map[code] || 'Something went wrong. Please try again.';
  }
</script>

<style>
:root {
  --bg: #0b0b12;
  --surface: #13121e;
  --surface2: #1c1a2e;
  --surface3: #252340;
  --border: #302d50;
  --accent: #f4a261;
  --accent2: #e76f51;
  --green: #06d6a0;
  --red: #ef476f;
  --blue: #4cc9f0;
  --purple: #b5a1e5;
  --text: #eae6ff;
  --text2: #a89ec9;
  --text3: #5e5880;
  --gold: #ffd166;
  --shadow: 0 12px 40px rgba(0,0,0,0.5);
  --radius: 14px;
  --radius-sm: 8px;
}
* { margin:0; padding:0; box-sizing:border-box; }
body { background:var(--bg); color:var(--text); font-family:'DM Sans',sans-serif; font-size:14px; min-height:100vh; }
::-webkit-scrollbar{width:5px}::-webkit-scrollbar-track{background:var(--surface)}::-webkit-scrollbar-thumb{background:var(--border);border-radius:3px}

/* ── AUTH SCREEN ── */
#authScreen {
  min-height:100vh; display:flex; align-items:center; justify-content:center;
  background: radial-gradient(ellipse at 20% 50%, rgba(244,162,97,0.06) 0%, transparent 60%),
              radial-gradient(ellipse at 80% 20%, rgba(76,201,240,0.05) 0%, transparent 50%),
              var(--bg);
}
.auth-box {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 22px;
  padding: 44px 40px;
  width: 420px;
  max-width: 95vw;
  box-shadow: var(--shadow);
  animation: authIn 0.4s cubic-bezier(.22,1,.36,1);
}
@keyframes authIn { from{opacity:0;transform:translateY(24px) scale(0.97)} to{opacity:1;transform:none} }
.auth-logo { text-align:center; margin-bottom:32px; }
.auth-logo-title { font-family:'Playfair Display',serif; font-size:28px; font-weight:700; color:var(--accent); }
.auth-logo-sub { font-size:11px; color:var(--text3); letter-spacing:2px; text-transform:uppercase; margin-top:4px; }
.auth-tab-bar { display:flex; gap:3px; background:var(--surface2); padding:3px; border-radius:var(--radius-sm); margin-bottom:24px; }
.auth-tab { flex:1; padding:8px; border-radius:6px; cursor:pointer; font-size:13px; font-weight:600; text-align:center; color:var(--text3); transition:all 0.18s; }
.auth-tab.active { background:var(--surface3); color:var(--text); }
.auth-panel { display:none; }
.auth-panel.active { display:block; }
.auth-field { margin-bottom:14px; }
.auth-field label { display:block; font-size:10px; letter-spacing:1px; text-transform:uppercase; color:var(--text3); margin-bottom:6px; }
.auth-field input {
  width:100%; background:var(--surface2); border:1px solid var(--border); border-radius:var(--radius-sm);
  color:var(--text); padding:11px 14px; font-size:14px; font-family:'DM Sans',sans-serif; outline:none; transition:border-color 0.2s;
}
.auth-field input:focus { border-color:var(--accent); }
.auth-error { background:rgba(239,71,111,0.1); border:1px solid rgba(239,71,111,0.3); border-radius:var(--radius-sm); padding:10px 14px; font-size:12px; color:var(--red); margin-bottom:14px; display:none; }
.auth-success { background:rgba(6,214,160,0.1); border:1px solid rgba(6,214,160,0.3); border-radius:var(--radius-sm); padding:10px 14px; font-size:12px; color:var(--green); margin-bottom:14px; display:none; }
.auth-btn {
  width:100%; padding:13px; border-radius:var(--radius-sm); border:none; cursor:pointer;
  background:linear-gradient(135deg,var(--accent),var(--accent2)); color:white;
  font-size:14px; font-weight:700; font-family:'DM Sans',sans-serif;
  transition:all 0.2s; margin-top:4px;
}
.auth-btn:hover:not(:disabled) { opacity:0.88; transform:translateY(-1px); }
.auth-btn:disabled { opacity:0.5; cursor:not-allowed; }
.auth-link { color:var(--accent); cursor:pointer; font-size:12px; text-decoration:underline; }
.auth-footer { text-align:center; margin-top:18px; font-size:12px; color:var(--text3); }

/* ── APP LAYOUT ── */
#appScreen { display:none; min-height:100vh; }
.app { display:flex; min-height:100vh; }

/* SIDEBAR */
.sidebar { width:220px; min-width:220px; background:var(--surface); border-right:1px solid var(--border); display:flex; flex-direction:column; padding:22px 0; position:sticky; top:0; height:100vh; overflow-y:auto; }
.logo { padding:0 18px 22px; border-bottom:1px solid var(--border); margin-bottom:14px; }
.logo-title { font-family:'Playfair Display',serif; font-size:21px; font-weight:700; color:var(--accent); }
.logo-sub { font-size:10px; color:var(--text3); letter-spacing:1.5px; text-transform:uppercase; margin-top:2px; }
.nav-section { padding:0 8px; margin-bottom:4px; }
.nav-label { font-size:10px; letter-spacing:1.5px; text-transform:uppercase; color:var(--text3); padding:0 10px; margin-bottom:3px; }
.nav-item { display:flex; align-items:center; gap:9px; padding:9px 10px; border-radius:var(--radius-sm); cursor:pointer; color:var(--text2); font-size:13px; font-weight:500; transition:all 0.18s; border:1px solid transparent; }
.nav-item:hover { background:var(--surface2); color:var(--text); }
.nav-item.active { background:linear-gradient(135deg,rgba(244,162,97,0.15),rgba(231,111,81,0.08)); color:var(--accent); border-color:rgba(244,162,97,0.25); }
.nav-icon { font-size:15px; width:18px; text-align:center; }
.sidebar-footer { margin-top:auto; padding:14px 14px 0; border-top:1px solid var(--border); }
.user-chip { background:var(--surface2); border:1px solid var(--border); border-radius:var(--radius-sm); padding:10px 12px; }
.user-chip-name { font-size:13px; font-weight:600; color:var(--text); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.user-chip-role { font-size:10px; letter-spacing:1px; text-transform:uppercase; margin-top:2px; }
.viewer-chip-role { color: var(--blue); }
.admin-chip-role { color: var(--green); }
.logout-btn { width:100%; margin-top:8px; padding:8px; background:rgba(239,71,111,0.08); border:1px solid rgba(239,71,111,0.2); border-radius:var(--radius-sm); color:var(--red); font-size:12px; font-weight:600; cursor:pointer; transition:all 0.18s; }
.logout-btn:hover { background:rgba(239,71,111,0.16); }

/* MAIN */
.main { flex:1; overflow-y:auto; overflow-x:hidden; }
.page { display:none; padding:28px; animation:fadeIn 0.25s ease; }
.page.active { display:block; }
@keyframes fadeIn{from{opacity:0;transform:translateY(6px)}to{opacity:1;transform:none}}

/* VIEW-ONLY BANNER */
.view-only-banner { background:rgba(76,201,240,0.08); border:1px solid rgba(76,201,240,0.25); border-radius:var(--radius-sm); padding:10px 16px; display:flex; align-items:center; gap:10px; margin-bottom:18px; font-size:13px; color:var(--blue); }

/* PAGE HEADER */
.page-header { display:flex; align-items:center; justify-content:space-between; margin-bottom:22px; flex-wrap:wrap; gap:10px; }
.page-title { font-family:'Playfair Display',serif; font-size:24px; font-weight:700; }
.page-subtitle { font-size:12px; color:var(--text3); margin-top:3px; }

/* CARDS */
.card { background:var(--surface); border:1px solid var(--border); border-radius:var(--radius); padding:20px; }
.grid-4 { display:grid; grid-template-columns:repeat(4,1fr); gap:14px; margin-bottom:18px; }
.grid-3 { display:grid; grid-template-columns:repeat(3,1fr); gap:14px; margin-bottom:18px; }
.grid-2 { display:grid; grid-template-columns:repeat(2,1fr); gap:18px; margin-bottom:18px; }

/* STAT CARDS */
.stat-card { background:var(--surface); border:1px solid var(--border); border-radius:var(--radius); padding:18px; position:relative; overflow:hidden; }
.stat-card::before { content:''; position:absolute; top:0; left:0; right:0; height:3px; }
.stat-card.green::before { background:linear-gradient(90deg,var(--green),#00b4d8); }
.stat-card.red::before { background:linear-gradient(90deg,var(--red),#f77f00); }
.stat-card.gold::before { background:linear-gradient(90deg,var(--gold),var(--accent)); }
.stat-card.blue::before { background:linear-gradient(90deg,var(--blue),var(--purple)); }
.stat-label { font-size:10px; letter-spacing:1px; text-transform:uppercase; color:var(--text3); margin-bottom:8px; }
.stat-value { font-family:'DM Mono',monospace; font-size:22px; font-weight:500; }
.stat-value.green{color:var(--green)}.stat-value.red{color:var(--red)}.stat-value.gold{color:var(--gold)}.stat-value.blue{color:var(--blue)}
.stat-sub { font-size:11px; color:var(--text3); margin-top:5px; }
.stat-icon { position:absolute; top:16px; right:16px; font-size:20px; opacity:0.3; }

/* BUTTONS */
.btn { padding:8px 16px; border-radius:var(--radius-sm); border:none; cursor:pointer; font-family:'DM Sans',sans-serif; font-size:13px; font-weight:600; transition:all 0.18s; display:inline-flex; align-items:center; gap:6px; }
.btn-primary { background:linear-gradient(135deg,var(--accent),var(--accent2)); color:white; }
.btn-primary:hover { opacity:0.88; transform:translateY(-1px); }
.btn-outline { background:transparent; border:1px solid var(--border); color:var(--text2); }
.btn-outline:hover { border-color:var(--accent); color:var(--accent); }
.btn-green { background:rgba(6,214,160,0.12); color:var(--green); border:1px solid rgba(6,214,160,0.25); }
.btn-red { background:rgba(239,71,111,0.1); color:var(--red); border:1px solid rgba(239,71,111,0.25); }
.btn-blue { background:rgba(76,201,240,0.1); color:var(--blue); border:1px solid rgba(76,201,240,0.25); }
.btn-sm { padding:5px 10px; font-size:11px; }
.btn-icon { padding:6px; width:30px; height:30px; justify-content:center; border-radius:6px; }
.btn:disabled { opacity:0.4; cursor:not-allowed; }

/* FORMS */
.form-row { display:grid; gap:12px; margin-bottom:12px; }
.form-row.cols-2{grid-template-columns:repeat(2,1fr)}.form-row.cols-3{grid-template-columns:repeat(3,1fr)}.form-row.cols-4{grid-template-columns:repeat(4,1fr)}
.form-group { display:flex; flex-direction:column; gap:5px; }
.form-group label { font-size:10px; letter-spacing:0.8px; text-transform:uppercase; color:var(--text3); }
.form-group input,.form-group select,.form-group textarea { background:var(--surface2); border:1px solid var(--border); border-radius:var(--radius-sm); color:var(--text); padding:8px 11px; font-family:'DM Sans',sans-serif; font-size:13px; outline:none; transition:border-color 0.2s; width:100%; }
.form-group input:focus,.form-group select:focus,.form-group textarea:focus { border-color:var(--accent); }
.form-group select option { background:var(--surface2); }
.form-actions { display:flex; gap:8px; margin-top:14px; justify-content:flex-end; }

/* QUICK ENTRY */
.type-toggle { display:flex; gap:4px; margin-bottom:14px; }
.type-btn { padding:8px 14px; border-radius:var(--radius-sm); border:1px solid var(--border); background:transparent; color:var(--text3); font-size:12px; font-weight:600; cursor:pointer; transition:all 0.18s; }
.type-btn.income.active { background:rgba(6,214,160,0.15); color:var(--green); border-color:rgba(6,214,160,0.3); }
.type-btn.expense.active { background:rgba(239,71,111,0.12); color:var(--red); border-color:rgba(239,71,111,0.3); }
.type-btn.transfer.active { background:rgba(76,201,240,0.1); color:var(--blue); border-color:rgba(76,201,240,0.3); }

/* TABLE */
.table-wrap { overflow-x:auto; }
table { width:100%; border-collapse:collapse; }
thead th { text-align:left; padding:9px 12px; font-size:10px; letter-spacing:1px; text-transform:uppercase; color:var(--text3); border-bottom:1px solid var(--border); white-space:nowrap; }
tbody td { padding:11px 12px; border-bottom:1px solid rgba(48,45,80,0.4); font-size:13px; color:var(--text2); }
tbody tr:hover td { background:rgba(255,255,255,0.02); color:var(--text); }
.amount-in{color:var(--green);font-family:'DM Mono',monospace;font-weight:500}
.amount-out{color:var(--red);font-family:'DM Mono',monospace;font-weight:500}
.amount-neutral{color:var(--gold);font-family:'DM Mono',monospace;font-weight:500}
.row-actions { display:flex; gap:5px; opacity:0; transition:opacity 0.18s; }
tbody tr:hover .row-actions { opacity:1; }

/* BADGES */
.badge { display:inline-flex; align-items:center; gap:4px; padding:2px 8px; border-radius:100px; font-size:11px; font-weight:600; }
.badge-green{background:rgba(6,214,160,0.12);color:var(--green)}
.badge-red{background:rgba(239,71,111,0.12);color:var(--red)}
.badge-gold{background:rgba(255,209,102,0.12);color:var(--gold)}
.badge-blue{background:rgba(76,201,240,0.12);color:var(--blue)}
.badge-purple{background:rgba(181,161,229,0.12);color:var(--purple)}
.badge-gray{background:rgba(255,255,255,0.06);color:var(--text3)}

/* PROGRESS */
.progress-wrap { background:var(--surface2); border-radius:100px; height:7px; overflow:hidden; }
.progress-bar { height:100%; border-radius:100px; transition:width 0.5s ease; }
.progress-bar.green{background:linear-gradient(90deg,var(--green),#00b4d8)}
.progress-bar.orange{background:linear-gradient(90deg,var(--accent),var(--accent2))}
.progress-bar.red{background:linear-gradient(90deg,var(--red),#f77f00)}

/* BUDGET CARD */
.budget-card { background:var(--surface); border:1px solid var(--border); border-radius:var(--radius); padding:16px; }
.budget-alert { margin-top:8px; padding:6px 10px; border-radius:6px; font-size:11px; font-weight:600; }
.budget-alert.warning { background:rgba(255,209,102,0.12); color:var(--gold); border:1px solid rgba(255,209,102,0.25); }
.budget-alert.danger { background:rgba(239,71,111,0.1); color:var(--red); border:1px solid rgba(239,71,111,0.25); }

/* ACCOUNT CARD */
.account-card { background:var(--surface); border:1px solid var(--border); border-radius:var(--radius); padding:18px; position:relative; overflow:hidden; transition:all 0.2s; cursor:pointer; }
.account-card:hover { border-color:var(--accent); transform:translateY(-2px); }
.account-deco { position:absolute; bottom:-15px; right:-15px; width:80px; height:80px; border-radius:50%; opacity:0.08; }
.account-type-badge { font-size:10px; letter-spacing:1px; text-transform:uppercase; margin-bottom:10px; }
.account-name { font-family:'Playfair Display',serif; font-size:16px; font-weight:600; }
.account-balance { font-family:'DM Mono',monospace; font-size:20px; margin-top:10px; }

/* SECTION TITLE */
.section-title { font-size:14px; font-weight:600; color:var(--text); margin-bottom:12px; display:flex; align-items:center; gap:8px; }
.section-title::after { content:''; flex:1; height:1px; background:var(--border); }

/* TABS */
.tabs { display:flex; gap:3px; background:var(--surface2); padding:3px; border-radius:var(--radius-sm); width:fit-content; margin-bottom:18px; flex-wrap:wrap; }
.tab { padding:6px 14px; border-radius:6px; cursor:pointer; font-size:12px; font-weight:500; color:var(--text3); transition:all 0.18s; }
.tab.active { background:var(--surface3); color:var(--text); }
.tab:hover:not(.active) { color:var(--text2); }

/* MODAL */
.modal-overlay { position:fixed; inset:0; background:rgba(0,0,0,0.7); display:flex; align-items:center; justify-content:center; z-index:100; opacity:0; pointer-events:none; transition:opacity 0.2s; padding:16px; }
.modal-overlay.open { opacity:1; pointer-events:all; }
.modal { background:var(--surface); border:1px solid var(--border); border-radius:18px; padding:26px; width:520px; max-width:100%; max-height:90vh; overflow-y:auto; transform:scale(0.95); transition:transform 0.2s; }
.modal-overlay.open .modal { transform:scale(1); }
.modal-header { display:flex; justify-content:space-between; align-items:center; margin-bottom:20px; }
.modal-title { font-family:'Playfair Display',serif; font-size:19px; font-weight:700; }
.modal-close { background:var(--surface2); border:1px solid var(--border); color:var(--text2); width:30px; height:30px; border-radius:50%; cursor:pointer; font-size:15px; display:flex; align-items:center; justify-content:center; transition:all 0.18s; }
.modal-close:hover { color:var(--red); border-color:var(--red); }
.confirm-dialog { background:var(--surface); border:1px solid rgba(239,71,111,0.4); border-radius:var(--radius); padding:28px; width:360px; max-width:100%; text-align:center; transform:scale(0.95); transition:transform 0.2s; }
.modal-overlay.open .confirm-dialog { transform:scale(1); }

/* TOAST */
.toast { position:fixed; bottom:24px; right:24px; background:var(--surface2); border:1px solid var(--border); border-radius:var(--radius); padding:12px 18px; font-size:13px; color:var(--text); z-index:300; display:flex; align-items:center; gap:9px; transform:translateX(130%); transition:transform 0.3s; box-shadow:var(--shadow); max-width:340px; }
.toast.show{transform:translateX(0)}.toast.success{border-color:var(--green)}.toast.error{border-color:var(--red)}.toast.info{border-color:var(--blue)}

/* LIVE INDICATOR */
.live-dot { width:7px; height:7px; background:var(--green); border-radius:50%; animation:pulse 2s infinite; }
@keyframes pulse{0%,100%{opacity:1;transform:scale(1)}50%{opacity:0.5;transform:scale(0.8)}}

/* USER CARDS */
.user-row { display:grid; grid-template-columns:1fr 80px 100px 80px; gap:10px; align-items:center; padding:12px 14px; border-bottom:1px solid rgba(48,45,80,0.4); }
.user-row:hover { background:rgba(255,255,255,0.02); }
.user-row.header { font-size:10px; text-transform:uppercase; letter-spacing:1px; color:var(--text3); border-radius:var(--radius-sm) var(--radius-sm) 0 0; }

/* EMPTY */
.empty-state { text-align:center; padding:44px 20px; color:var(--text3); }
.empty-state .icon { font-size:36px; margin-bottom:10px; opacity:0.5; }

/* OVERDRAFT BANNER */
.overdraft-banner { background:rgba(239,71,111,0.1); border:1px solid rgba(239,71,111,0.3); border-radius:var(--radius-sm); padding:11px 16px; display:flex; align-items:center; gap:10px; margin-bottom:12px; font-size:13px; color:var(--red); }

/* SEARCH */
.search-bar { display:flex; align-items:center; gap:10px; margin-bottom:16px; flex-wrap:wrap; }
.search-wrap { position:relative; flex:1; min-width:180px; }
.search-wrap input { padding-left:32px; }
.search-icon-inp { position:absolute; left:10px; top:50%; transform:translateY(-50%); color:var(--text3); font-size:14px; pointer-events:none; }

/* UTILS */
.flex{display:flex}.flex-between{display:flex;justify-content:space-between;align-items:center}
.flex-center{display:flex;align-items:center;gap:8px}
.mt-1{margin-top:8px}.mt-2{margin-top:16px}.mt-3{margin-top:24px}
.mb-1{margin-bottom:8px}.mb-2{margin-bottom:16px}
.text-sm{font-size:12px}.text-xs{font-size:11px}.text-muted{color:var(--text3)}
.text-green{color:var(--green)}.text-red{color:var(--red)}.text-gold{color:var(--gold)}.text-blue{color:var(--blue)}
.font-mono{font-family:'DM Mono',monospace}.fw-600{font-weight:600}.w-full{width:100%}

/* SETUP SCREEN */
.setup-box { background:var(--surface); border:1px solid rgba(244,162,97,0.3); border-radius:22px; padding:40px; width:480px; max-width:95vw; }
.setup-steps { counter-reset:step; }
.setup-step { counter-increment:step; margin-bottom:10px; padding:12px 16px; background:var(--surface2); border-radius:var(--radius-sm); font-size:13px; color:var(--text2); position:relative; padding-left:42px; }
.setup-step::before { content:counter(step); position:absolute; left:14px; top:50%; transform:translateY(-50%); background:var(--accent); color:white; width:20px; height:20px; border-radius:50%; font-size:11px; font-weight:700; display:flex; align-items:center; justify-content:center; }

@media(max-width:1100px){.grid-4{grid-template-columns:repeat(2,1fr)}.grid-3{grid-template-columns:repeat(2,1fr)}}
@media(max-width:768px){.sidebar{width:52px;min-width:52px}.logo-title,.logo-sub,.nav-label,.nav-item span{display:none}.nav-item{justify-content:center;padding:10px}.page{padding:16px 12px}.grid-4,.grid-3,.grid-2{grid-template-columns:1fr 1fr}}
</style>
</head>
<body>

<!-- ══════════════ AUTH SCREEN ══════════════ -->
<div id="authScreen">
  <div id="authBoxWrap">
    <!-- SETUP PROMPT (shown if Firebase not configured) -->
    <div id="setupPrompt" class="setup-box" style="display:none">
      <div class="auth-logo">
        <div class="auth-logo-title">GharKhata</div>
        <div class="auth-logo-sub">Setup Required</div>
      </div>
      <p style="font-size:13px;color:var(--text2);margin-bottom:20px">To enable multi-device access with login, you need to connect a free Firebase project. Follow these steps:</p>
      <div class="setup-steps">
        <div class="setup-step">Go to <strong>console.firebase.google.com</strong> and create a new project</div>
        <div class="setup-step">Enable <strong>Authentication → Email/Password</strong> sign-in method</div>
        <div class="setup-step">Enable <strong>Firestore Database</strong> (start in test mode)</div>
        <div class="setup-step">Go to <strong>Project Settings → General → Your apps → Web app</strong> → copy the config</div>
        <div class="setup-step">Open this file and replace the <strong>firebaseConfig</strong> object at the top with your config</div>
        <div class="setup-step">The <strong>first user to sign up</strong> automatically becomes Admin</div>
      </div>
      <div style="margin-top:20px;padding:14px;background:var(--surface2);border-radius:var(--radius-sm);font-family:'DM Mono',monospace;font-size:11px;color:var(--text3)">
        💡 Firestore Security Rules: In Firebase console → Firestore → Rules, paste the rules from <strong>firestore.rules</strong> file included.
      </div>
    </div>

    <!-- LOGIN BOX -->
    <div class="auth-box" id="loginBox">
      <div class="auth-logo">
        <div class="auth-logo-title">GharKhata</div>
        <div class="auth-logo-sub">Household Finance Portal</div>
      </div>
      <div class="auth-tab-bar">
        <div class="auth-tab active" id="tab-login" onclick="switchAuthTab('login')">Sign In</div>
        <div class="auth-tab" id="tab-reset" onclick="switchAuthTab('reset')">Forgot Password</div>
      </div>

      <!-- LOGIN -->
      <div class="auth-panel active" id="panel-login">
        <div class="auth-error" id="login-error"></div>
        <div class="auth-field"><label>Email Address</label><input id="login-email" type="email" placeholder="you@example.com" onkeydown="if(event.key==='Enter')doLogin()" /></div>
        <div class="auth-field">
          <label>Password</label>
          <div style="position:relative">
            <input id="login-pass" type="password" placeholder="••••••••" onkeydown="if(event.key==='Enter')doLogin()" style="padding-right:44px" />
            <button onclick="togglePassVis('login-pass',this)" style="position:absolute;right:10px;top:50%;transform:translateY(-50%);background:none;border:none;cursor:pointer;color:var(--text3);font-size:16px">👁</button>
          </div>
        </div>
        <button class="auth-btn" id="login-btn" onclick="doLogin()">Sign In →</button>
        <div class="auth-footer">Don't have access? <span class="auth-link" onclick="switchAuthTab('reset')">Contact admin or reset password</span></div>
      </div>

      <!-- RESET -->
      <div class="auth-panel" id="panel-reset">
        <div class="auth-error" id="reset-error"></div>
        <div class="auth-success" id="reset-success"></div>
        <p style="font-size:13px;color:var(--text2);margin-bottom:18px">Enter your registered email and we'll send a password reset link.</p>
        <div class="auth-field"><label>Email Address</label><input id="reset-email" type="email" placeholder="you@example.com" onkeydown="if(event.key==='Enter')doResetPassword()" /></div>
        <button class="auth-btn" onclick="doResetPassword()">Send Reset Email →</button>
        <div class="auth-footer"><span class="auth-link" onclick="switchAuthTab('login')">← Back to Sign In</span></div>
      </div>
    </div>
  </div>
</div>

<!-- ══════════════ APP SCREEN ══════════════ -->
<div id="appScreen">
<div class="app">

<!-- SIDEBAR -->
<nav class="sidebar">
  <div class="logo">
    <div class="logo-title">GharKhata</div>
    <div style="display:flex;align-items:center;gap:6px;margin-top:4px"><div class="live-dot"></div><div class="logo-sub" style="margin-top:0">Live Sync</div></div>
  </div>
  <div class="nav-section">
    <div class="nav-label">Overview</div>
    <div class="nav-item active" onclick="navigate('dashboard')"><span class="nav-icon">🏠</span><span>Dashboard</span></div>
    <div class="nav-item" onclick="navigate('summary')"><span class="nav-icon">📊</span><span>Reports</span></div>
  </div>
  <div class="nav-section" style="margin-top:6px">
    <div class="nav-label">Transactions</div>
    <div class="nav-item" onclick="navigate('transactions')"><span class="nav-icon">📋</span><span>All Entries</span></div>
    <div class="nav-item" onclick="navigate('income')"><span class="nav-icon">💰</span><span>Income</span></div>
    <div class="nav-item" onclick="navigate('expenses')"><span class="nav-icon">🧾</span><span>Expenses</span></div>
  </div>
  <div class="nav-section" style="margin-top:6px">
    <div class="nav-label">Planning</div>
    <div class="nav-item" onclick="navigate('budget')"><span class="nav-icon">🎯</span><span>Budget Limits</span></div>
    <div class="nav-item" onclick="navigate('accounts')"><span class="nav-icon">🏦</span><span>Accounts</span></div>
  </div>
  <div class="nav-section" style="margin-top:6px">
    <div class="nav-label">Liabilities</div>
    <div class="nav-item" onclick="navigate('loans')"><span class="nav-icon">💳</span><span>Loans & EMI</span></div>
  </div>
  <div class="nav-section" id="adminNavSection" style="margin-top:6px;display:none">
    <div class="nav-label">Admin</div>
    <div class="nav-item" onclick="navigate('users')"><span class="nav-icon">👥</span><span>Users</span></div>
  </div>
  <div class="sidebar-footer">
    <div class="user-chip">
      <div class="user-chip-name" id="sidebarUserName">—</div>
      <div class="user-chip-role" id="sidebarUserRole">—</div>
    </div>
    <button class="logout-btn" onclick="doLogout()">Sign Out</button>
  </div>
</nav>

<!-- MAIN CONTENT -->
<main class="main">

<!-- ─── DASHBOARD ─── -->
<div class="page active" id="page-dashboard">
  <div class="page-header">
    <div><div class="page-title">Financial Dashboard</div><div class="page-subtitle">Real-time household money overview</div></div>
    <button class="btn btn-primary btn-sm" id="dashQuickBtn" onclick="openQuickEntry()">⚡ Quick Add</button>
  </div>
  <div id="dashViewBanner"></div>
  <div id="dashOverdraftAlerts"></div>
  <div class="grid-4">
    <div class="stat-card green"><div class="stat-icon">💵</div><div class="stat-label">Monthly Income</div><div class="stat-value green" id="dash-income">₹0</div><div class="stat-sub">This month</div></div>
    <div class="stat-card red"><div class="stat-icon">🧾</div><div class="stat-label">Total Expenses</div><div class="stat-value red" id="dash-expense">₹0</div><div class="stat-sub">This month</div></div>
    <div class="stat-card gold"><div class="stat-icon">💎</div><div class="stat-label">Net Savings</div><div class="stat-value gold" id="dash-savings">₹0</div><div class="stat-sub">Income − expenses</div></div>
    <div class="stat-card blue"><div class="stat-icon">🏦</div><div class="stat-label">Total Assets</div><div class="stat-value blue" id="dash-assets">₹0</div><div class="stat-sub">All accounts</div></div>
  </div>
  <div class="grid-2">
    <div class="card"><div class="section-title">🎯 Budget Status</div><div id="dashBudgetCards" style="display:flex;flex-direction:column;gap:10px"></div><button class="btn btn-outline btn-sm mt-2 w-full" onclick="navigate('budget')">Manage Budgets →</button></div>
    <div class="card"><div class="flex-between mb-2"><div class="section-title" style="margin-bottom:0">🕐 Recent Entries</div><button class="btn btn-outline btn-sm" onclick="navigate('transactions')">View All</button></div><div id="dashRecentTxns"></div></div>
  </div>
  <div class="section-title mt-2">🏦 Account Balances</div>
  <div class="grid-3" id="dashAccounts"></div>
</div>

<!-- ─── ALL TRANSACTIONS ─── -->
<div class="page" id="page-transactions">
  <div class="page-header">
    <div><div class="page-title">All Entries</div><div class="page-subtitle">View, search, edit and manage every transaction</div></div>
    <button class="btn btn-primary btn-sm" id="txnAddBtn" onclick="openQuickEntry()">⚡ Quick Add</button>
  </div>
  <div id="txnViewBanner"></div>

  <!-- QUICK ENTRY -->
  <div class="card mb-2" id="quickEntrySection" style="display:none">
    <div class="section-title">⚡ Quick Entry <span class="text-xs text-muted font-mono" style="font-weight:400">— All fields in one go</span></div>
    <div class="type-toggle">
      <button class="type-btn income active" id="qtype-income" onclick="setQType('income')">+ Income</button>
      <button class="type-btn expense" id="qtype-expense" onclick="setQType('expense')">− Expense</button>
      <button class="type-btn transfer" id="qtype-transfer" onclick="setQType('transfer')">⇄ Transfer</button>
    </div>
    <div class="form-row cols-4">
      <div class="form-group"><label>Description *</label><input id="q-desc" placeholder="What was this?" /></div>
      <div class="form-group"><label>Amount ₹ *</label><input id="q-amount" type="number" min="0" step="0.01" placeholder="0.00" /></div>
      <div class="form-group"><label>Category</label><select id="q-category"></select></div>
      <div class="form-group"><label>Account *</label><select id="q-account"><option value="">Select…</option></select></div>
    </div>
    <div id="q-transfer-fields" style="display:none">
      <div class="form-row cols-3">
        <div class="form-group"><label>To Account *</label><select id="q-to-account"><option value="">Select…</option></select></div>
        <div class="form-group"><label>Date *</label><input id="q-date-t" type="date" /></div>
        <div class="form-group"><label>Note</label><input id="q-note-t" placeholder="Optional note" /></div>
      </div>
    </div>
    <div id="q-normal-fields">
      <div class="form-row cols-2">
        <div class="form-group"><label>Date *</label><input id="q-date" type="date" /></div>
        <div class="form-group"><label>Note</label><input id="q-note" placeholder="Optional note" /></div>
      </div>
    </div>
    <div class="form-actions">
      <button class="btn btn-outline btn-sm" onclick="closeQuickEntry()">✕ Cancel</button>
      <button class="btn btn-primary btn-sm" onclick="saveQuickEntry()" id="q-save-btn">✓ Save Entry</button>
    </div>
  </div>

  <!-- FILTERS -->
  <div class="search-bar">
    <div class="search-wrap"><span class="search-icon-inp">🔍</span><input placeholder="Search description, category…" oninput="filterTxns()" id="txn-search" /></div>
    <select onchange="filterTxns()" id="txn-type-filter" style="background:var(--surface2);border:1px solid var(--border);border-radius:var(--radius-sm);color:var(--text);padding:8px 11px;font-size:13px;outline:none">
      <option value="">All Types</option><option value="income">Income</option><option value="expense">Expense</option><option value="transfer">Transfer</option>
    </select>
    <select onchange="filterTxns()" id="txn-cat-filter" style="background:var(--surface2);border:1px solid var(--border);border-radius:var(--radius-sm);color:var(--text);padding:8px 11px;font-size:13px;outline:none">
      <option value="">All Categories</option>
    </select>
  </div>
  <div class="card">
    <div class="table-wrap"><table>
      <thead><tr><th>Date</th><th>Description</th><th>Category</th><th>Account</th><th>Amount</th><th>Type</th><th style="text-align:right">Actions</th></tr></thead>
      <tbody id="txnBody"></tbody>
    </table></div>
    <div id="txnEmpty" class="empty-state" style="display:none"><div class="icon">📋</div><p>No transactions yet.</p></div>
  </div>
</div>

<!-- ─── INCOME ─── -->
<div class="page" id="page-income">
  <div class="page-header"><div><div class="page-title">Income</div><div class="page-subtitle">All income entries</div></div><button class="btn btn-primary btn-sm" id="incAddBtn" onclick="openQuickEntry('income')">+ Add Income</button></div>
  <div class="card"><div class="table-wrap"><table><thead><tr><th>Date</th><th>Description</th><th>Category</th><th>Account</th><th>Amount</th><th style="text-align:right">Actions</th></tr></thead><tbody id="incBody"></tbody></table></div><div id="incEmpty" class="empty-state" style="display:none"><div class="icon">💰</div><p>No income recorded yet.</p></div></div>
</div>

<!-- ─── EXPENSES ─── -->
<div class="page" id="page-expenses">
  <div class="page-header"><div><div class="page-title">Expenses</div><div class="page-subtitle">All expense entries</div></div><button class="btn btn-primary btn-sm" id="expAddBtn" onclick="openQuickEntry('expense')">+ Add Expense</button></div>
  <div class="card"><div class="table-wrap"><table><thead><tr><th>Date</th><th>Description</th><th>Category</th><th>Account</th><th>Amount</th><th style="text-align:right">Actions</th></tr></thead><tbody id="expBody"></tbody></table></div><div id="expEmpty" class="empty-state" style="display:none"><div class="icon">🧾</div><p>No expenses recorded yet.</p></div>
</div></div>

<!-- ─── BUDGET ─── -->
<div class="page" id="page-budget">
  <div class="page-header"><div><div class="page-title">Budget Limits</div><div class="page-subtitle">Set limits per category with overdraft alerts</div></div><button class="btn btn-primary btn-sm" id="budAddBtn" onclick="openAddBudget()">+ Set Budget</button></div>
  <div id="budgetAlerts" style="margin-bottom:16px"></div>
  <div class="grid-3" id="budgetGrid"><div class="empty-state" style="grid-column:1/-1"><div class="icon">🎯</div><p>No budgets set yet.</p></div></div>
</div>

<!-- ─── ACCOUNTS ─── -->
<div class="page" id="page-accounts">
  <div class="page-header"><div><div class="page-title">Accounts</div><div class="page-subtitle">All bank accounts, wallets and cash</div></div><button class="btn btn-primary btn-sm" id="accAddBtn" onclick="openAddAccount()">+ Add Account</button></div>
  <div class="grid-3" id="accountsGrid"></div>
  <div class="section-title mt-3">📋 Account-wise Transactions</div>
  <div class="tabs" id="accountTabs"></div>
  <div class="card"><div class="table-wrap"><table><thead><tr><th>Date</th><th>Description</th><th>Category</th><th>Amount</th><th>Type</th><th style="text-align:right">Actions</th></tr></thead><tbody id="accTxnBody"></tbody></table></div><div id="accTxnEmpty" class="empty-state" style="display:none"><div class="icon">🏦</div><p>No transactions for this account.</p></div></div>
</div>

<!-- ─── LOANS ─── -->
<div class="page" id="page-loans">
  <div class="page-header"><div><div class="page-title">Loans & EMI</div><div class="page-subtitle">Track all liabilities</div></div><button class="btn btn-primary btn-sm" id="loanAddBtn" onclick="openModal('addLoanModal')">+ Add Loan</button></div>
  <div class="grid-2" id="loansGrid"><div class="empty-state" style="grid-column:1/-1"><div class="icon">💳</div><p>No loans added yet.</p></div></div>
</div>

<!-- ─── USERS (ADMIN) ─── -->
<div class="page" id="page-users">
  <div class="page-header"><div><div class="page-title">User Management</div><div class="page-subtitle">Control who can access GharKhata</div></div><button class="btn btn-primary btn-sm" onclick="openModal('addUserModal')">+ Add User</button></div>
  <div class="card">
    <div class="user-row header"><span>User</span><span>Role</span><span>Change Role</span><span>Remove</span></div>
    <div id="usersTable"></div>
  </div>
  <div class="card mt-2" style="background:rgba(76,201,240,0.05);border-color:rgba(76,201,240,0.2)">
    <div class="section-title" style="color:var(--blue)">ℹ️ Access Levels</div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px">
      <div><div class="badge badge-green mb-1">Admin</div><div class="text-sm text-muted">Full access — add, edit, delete transactions, accounts, budgets. Manage users.</div></div>
      <div><div class="badge badge-blue mb-1">Viewer</div><div class="text-sm text-muted">Read-only access — can see all data but cannot make any changes.</div></div>
    </div>
  </div>
</div>

<!-- ─── SUMMARY ─── -->
<div class="page" id="page-summary">
  <div class="page-header"><div><div class="page-title">Financial Report</div><div class="page-subtitle">Monthly summary and analysis</div></div></div>
  <div class="grid-4" id="summaryStats"></div>
  <div class="grid-2">
    <div class="card"><div class="section-title">📈 Income by Category</div><div id="summaryInc"></div></div>
    <div class="card"><div class="section-title">📉 Expenses by Category</div><div id="summaryExp"></div></div>
  </div>
  <div class="card mt-2"><div class="section-title">📋 All Transactions This Month</div><div class="table-wrap"><table><thead><tr><th>Date</th><th>Description</th><th>Category</th><th>Account</th><th>Amount</th><th>Type</th></tr></thead><tbody id="summaryBody"></tbody></table></div></div>
</div>

</main>
</div>
</div><!-- appScreen -->

<!-- ════════════ MODALS ════════════ -->

<!-- EDIT TRANSACTION -->
<div class="modal-overlay" id="editTxnModal">
  <div class="modal">
    <div class="modal-header"><div class="modal-title">✏️ Edit Transaction</div><button class="modal-close" onclick="closeModal('editTxnModal')">✕</button></div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Type</label><select id="et-type" onchange="updateEditCats()"><option value="income">Income</option><option value="expense">Expense</option><option value="transfer">Transfer</option></select></div>
      <div class="form-group"><label>Date *</label><input id="et-date" type="date" /></div>
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Description *</label><input id="et-desc" /></div>
      <div class="form-group"><label>Amount ₹ *</label><input id="et-amount" type="number" /></div>
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Category</label><select id="et-category"></select></div>
      <div class="form-group"><label>Account *</label><select id="et-account"></select></div>
    </div>
    <div class="form-row" id="et-to-row" style="display:none">
      <div class="form-group"><label>To Account</label><select id="et-to-account"></select></div>
    </div>
    <div class="form-row"><div class="form-group"><label>Note</label><input id="et-note" placeholder="Optional" /></div></div>
    <input type="hidden" id="et-id" />
    <div class="form-actions">
      <button class="btn btn-outline btn-sm" onclick="closeModal('editTxnModal')">Cancel</button>
      <button class="btn btn-primary btn-sm" onclick="saveEditTxn()">✓ Save Changes</button>
    </div>
  </div>
</div>

<!-- ADD BUDGET -->
<div class="modal-overlay" id="addBudgetModal">
  <div class="modal">
    <div class="modal-header"><div class="modal-title" id="budModalTitle">🎯 Set Budget Limit</div><button class="modal-close" onclick="closeModal('addBudgetModal')">✕</button></div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Category *</label><select id="bud-cat"></select></div>
      <div class="form-group"><label>Monthly Limit ₹ *</label><input id="bud-limit" type="number" placeholder="e.g. 10000" /></div>
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Warn at (%)</label><input id="bud-warn" type="number" value="80" min="1" max="100" /></div>
      <div class="form-group"><label>Overdraft policy</label><select id="bud-overdraft"><option value="no">Block if exceeded</option><option value="yes">Allow but alert</option></select></div>
    </div>
    <input type="hidden" id="bud-edit-id" />
    <div class="form-actions">
      <button class="btn btn-outline btn-sm" onclick="closeModal('addBudgetModal')">Cancel</button>
      <button class="btn btn-primary btn-sm" onclick="saveBudget()">✓ Save Budget</button>
    </div>
  </div>
</div>

<!-- ADD ACCOUNT -->
<div class="modal-overlay" id="addAccountModal">
  <div class="modal">
    <div class="modal-header"><div class="modal-title" id="accModalTitle">🏦 Add Account</div><button class="modal-close" onclick="closeModal('addAccountModal')">✕</button></div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Account Name *</label><input id="acc-name" placeholder="e.g. SBI Savings" /></div>
      <div class="form-group"><label>Type</label><select id="acc-type"><option value="savings">Savings Bank</option><option value="current">Current Account</option><option value="wallet">Digital Wallet</option><option value="cash">Cash</option><option value="fd">Fixed Deposit</option><option value="ppf">PPF / Investment</option></select></div>
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Opening Balance ₹</label><input id="acc-balance" type="number" value="0" /></div>
      <div class="form-group"><label>Color</label><select id="acc-color"><option value="blue">Blue</option><option value="green">Green</option><option value="gold">Gold</option><option value="purple">Purple</option><option value="red">Red</option></select></div>
    </div>
    <div class="form-row"><div class="form-group"><label>Note (optional)</label><input id="acc-note" placeholder="Branch, last 4 digits, etc." /></div></div>
    <input type="hidden" id="acc-edit-id" />
    <div class="form-actions">
      <button class="btn btn-outline btn-sm" onclick="closeModal('addAccountModal')">Cancel</button>
      <button class="btn btn-primary btn-sm" onclick="saveAccount()">✓ Save Account</button>
    </div>
  </div>
</div>

<!-- ADD LOAN -->
<div class="modal-overlay" id="addLoanModal">
  <div class="modal">
    <div class="modal-header"><div class="modal-title">💳 Add Loan / EMI</div><button class="modal-close" onclick="closeModal('addLoanModal')">✕</button></div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Loan Name *</label><input id="loan-name" placeholder="e.g. Home Loan" /></div>
      <div class="form-group"><label>Lender</label><input id="loan-lender" placeholder="e.g. SBI Bank" /></div>
    </div>
    <div class="form-row cols-3">
      <div class="form-group"><label>Principal ₹</label><input id="loan-principal" type="number" /></div>
      <div class="form-group"><label>Rate (%)</label><input id="loan-rate" type="number" step="0.01" /></div>
      <div class="form-group"><label>Tenure (months)</label><input id="loan-tenure" type="number" /></div>
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>EMI ₹</label><input id="loan-emi" type="number" /></div>
      <div class="form-group"><label>Next Due Date</label><input id="loan-due" type="date" /></div>
    </div>
    <div class="form-actions">
      <button class="btn btn-outline btn-sm" onclick="closeModal('addLoanModal')">Cancel</button>
      <button class="btn btn-primary btn-sm" onclick="saveLoan()">✓ Save</button>
    </div>
  </div>
</div>

<!-- ADD USER (ADMIN) -->
<div class="modal-overlay" id="addUserModal">
  <div class="modal">
    <div class="modal-header"><div class="modal-title">👤 Add New User</div><button class="modal-close" onclick="closeModal('addUserModal')">✕</button></div>
    <div style="background:rgba(76,201,240,0.08);border:1px solid rgba(76,201,240,0.2);border-radius:var(--radius-sm);padding:10px 14px;font-size:12px;color:var(--blue);margin-bottom:16px">
      💡 The new user will receive a login email and can use "Forgot Password" to set their own password. A password reset email is optional — just provide a temporary password they can change.
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Full Name *</label><input id="new-user-name" placeholder="e.g. Priya Sharma" /></div>
      <div class="form-group"><label>Email *</label><input id="new-user-email" type="email" placeholder="priya@example.com" /></div>
    </div>
    <div class="form-row cols-2">
      <div class="form-group"><label>Temporary Password *</label><input id="new-user-pass" type="password" placeholder="Min 6 characters" /></div>
      <div class="form-group"><label>Role</label><select id="new-user-role"><option value="viewer">Viewer (read-only)</option><option value="admin">Admin (full access)</option></select></div>
    </div>
    <div class="form-actions">
      <button class="btn btn-outline btn-sm" onclick="closeModal('addUserModal')">Cancel</button>
      <button class="btn btn-primary btn-sm" onclick="doCreateUser()">✓ Create User</button>
    </div>
  </div>
</div>

<!-- CONFIRM DELETE -->
<div class="modal-overlay" id="confirmModal">
  <div class="confirm-dialog">
    <div style="font-size:36px;margin-bottom:12px">🗑️</div>
    <div style="font-size:16px;font-weight:700;margin-bottom:8px">Delete this entry?</div>
    <div style="font-size:13px;color:var(--text3);margin-bottom:20px" id="confirmMsg">This cannot be undone.</div>
    <div style="display:flex;gap:10px;justify-content:center">
      <button class="btn btn-outline btn-sm" onclick="closeModal('confirmModal')">Cancel</button>
      <button class="btn btn-red btn-sm" onclick="doConfirmDelete()">Delete</button>
    </div>
  </div>
</div>

<!-- TOAST -->
<div class="toast" id="toast"></div>

<script>
// ══════════════════════════════════════════════════
// CONSTANTS
// ══════════════════════════════════════════════════
const INCOME_CATS = ['Salary','Business','Freelance','Rental Income','Investment Returns','Bonus','Gift','Other Income'];
const EXPENSE_CATS = ['Groceries','Vegetables & Fruits','Milk & Dairy','Rent','Electricity','Water','Gas','Mobile / Internet','Medicines','Doctor / Hospital','School Fees','Petrol / Fuel','Vehicle Maintenance','Eating Out','Movies / OTT','Shopping / Clothes','Household Items','Repairs & Maintenance','EMI Payment','Insurance','Domestic Help','Donations','Other Expense'];
const COLORS = { green:'#06d6a0', blue:'#4cc9f0', gold:'#ffd166', purple:'#b5a1e5', red:'#ef476f' };
const ACC_TYPE_LABELS = { savings:'Savings Bank', current:'Current', wallet:'Digital Wallet', cash:'Cash', fd:'Fixed Deposit', ppf:'PPF/Investment' };

let currentQType = 'income';
let deleteTarget = null, deleteType = null;
let activeAccTab = null;
let currentPage = 'dashboard';

// ══ AUTH UI ══
function showAuth() {
  document.getElementById('authScreen').style.display='flex';
  document.getElementById('appScreen').style.display='none';
  // Check if Firebase is configured
  if(typeof window.GK === 'undefined') {
    document.getElementById('loginBox').style.display='none';
    document.getElementById('setupPrompt').style.display='block';
  }
}
function showApp() {
  document.getElementById('authScreen').style.display='none';
  document.getElementById('appScreen').style.display='block';
  const isAdmin = window.GK.role === 'admin';
  document.getElementById('sidebarUserName').textContent = window.GK.displayName || 'User';
  document.getElementById('sidebarUserRole').textContent = isAdmin ? '🛡️ Admin' : '👁️ Viewer';
  document.getElementById('sidebarUserRole').className = 'user-chip-role ' + (isAdmin ? 'admin-chip-role' : 'viewer-chip-role');
  document.getElementById('adminNavSection').style.display = isAdmin ? 'block' : 'none';
  // Admin-only buttons
  ['dashQuickBtn','txnAddBtn','incAddBtn','expAddBtn','budAddBtn','accAddBtn','loanAddBtn'].forEach(id => {
    const el = document.getElementById(id);
    if(el) el.style.display = isAdmin ? '' : 'none';
  });
  navigate('dashboard');
}

function switchAuthTab(tab) {
  ['login','reset'].forEach(t => {
    document.getElementById('tab-'+t).classList.toggle('active', t===tab);
    document.getElementById('panel-'+t).classList.toggle('active', t===tab);
  });
  clearAuthErrors();
}
function setAuthError(msg) {
  const panel = document.querySelector('.auth-panel.active').id.replace('panel-','');
  const el = document.getElementById(panel+'-error');
  el.textContent = msg; el.style.display='block';
}
function clearAuthErrors() {
  document.querySelectorAll('.auth-error,.auth-success').forEach(e=>{ e.style.display='none'; e.textContent=''; });
}
function togglePassVis(inputId, btn) {
  const inp = document.getElementById(inputId);
  inp.type = inp.type==='password' ? 'text' : 'password';
  btn.textContent = inp.type==='password' ? '👁' : '🙈';
}

// ══ NAVIGATE ══
function navigate(page) {
  currentPage = page;
  document.querySelectorAll('.page').forEach(p=>p.classList.remove('active'));
  document.querySelectorAll('.nav-item').forEach(n=>n.classList.remove('active'));
  const pageEl = document.getElementById('page-'+page);
  if(pageEl) pageEl.classList.add('active');
  document.querySelectorAll('.nav-item').forEach(n=>{
    const oc = n.getAttribute('onclick')||'';
    if(oc.includes("'"+page+"'")) n.classList.add('active');
  });
  renderPage(page);
}

function refreshCurrentPage() { renderPage(currentPage); }

function renderPage(page) {
  const G = window.GK;
  if(!G||!G.user) return;
  if(page==='dashboard') renderDashboard();
  else if(page==='transactions') renderTransactions();
  else if(page==='income') renderTypeTable('income');
  else if(page==='expenses') renderTypeTable('expense');
  else if(page==='budget') renderBudget();
  else if(page==='accounts') renderAccounts();
  else if(page==='loans') renderLoans();
  else if(page==='summary') renderSummary();
  else if(page==='users') renderUsers();
}

// ══ HELPERS ══
function fmt(n) { return '₹'+Number(n||0).toLocaleString('en-IN'); }
function today() { return new Date().toISOString().split('T')[0]; }
function isAdmin() { return window.GK && window.GK.role==='admin'; }
function G() { return window.GK; }

function thisMonthTxns() {
  const n=new Date(),m=n.getMonth(),y=n.getFullYear();
  return (G().transactions||[]).filter(t=>{ const d=new Date(t.date); return d.getMonth()===m&&d.getFullYear()===y; });
}
function monthlyIncome() { return thisMonthTxns().filter(t=>t.type==='income').reduce((s,t)=>s+Number(t.amount),0); }
function monthlyExpense() { return thisMonthTxns().filter(t=>t.type==='expense').reduce((s,t)=>s+Number(t.amount),0); }
function totalAssets() { return (G().accounts||[]).reduce((s,a)=>s+Number(a.balance||0),0); }
function categorySpend(cat) { return thisMonthTxns().filter(t=>t.type==='expense'&&t.category===cat).reduce((s,t)=>s+Number(t.amount),0); }
function accName(id) { const a=(G().accounts||[]).find(a=>a.id===id); return a?a.name:'—'; }

function viewOnlyBanner() {
  return isAdmin() ? '' : '<div class="view-only-banner">👁️ <strong>View-only mode.</strong> You can browse all data but cannot make changes. Contact the admin to edit.</div>';
}

// ══ DASHBOARD ══
function renderDashboard() {
  document.getElementById('dash-income').textContent=fmt(monthlyIncome());
  document.getElementById('dash-expense').textContent=fmt(monthlyExpense());
  const sav=monthlyIncome()-monthlyExpense();
  const sel=document.getElementById('dash-savings');
  sel.textContent=fmt(sav); sel.className='stat-value '+(sav>=0?'gold':'red');
  document.getElementById('dash-assets').textContent=fmt(totalAssets());
  document.getElementById('dashViewBanner').innerHTML=viewOnlyBanner();

  // Overdraft alerts
  const alerts=getBudgetAlerts().filter(a=>a.type==='overdraft'||a.pct>=a.warnAt);
  document.getElementById('dashOverdraftAlerts').innerHTML=alerts.map(a=>`<div class="overdraft-banner">⚠️ <strong>${a.cat}:</strong> ${a.msg}</div>`).join('');

  // Budget cards
  const bc=document.getElementById('dashBudgetCards');
  if(!G().budgets||G().budgets.length===0){ bc.innerHTML='<div class="text-muted text-sm">No budgets set.</div>'; }
  else bc.innerHTML=G().budgets.slice(0,4).map(b=>{
    const spent=categorySpend(b.category),pct=Math.min((spent/b.limit)*100,100),over=spent>b.limit;
    return `<div><div class="flex-between mb-1"><span style="font-size:13px;font-weight:600">${b.category}</span><span class="font-mono text-sm ${over?'text-red':'text-muted'}">${fmt(spent)} / ${fmt(b.limit)}</span></div><div class="progress-wrap"><div class="progress-bar ${over?'red':pct>b.warnAt?'orange':'green'}" style="width:${Math.min(pct,100)}%"></div></div></div>`;
  }).join('');

  // Recent
  const recent=[...(G().transactions||[])].sort((a,b)=>new Date(b.date)-new Date(a.date)).slice(0,6);
  const rct=document.getElementById('dashRecentTxns');
  if(recent.length===0){ rct.innerHTML='<div class="text-muted text-sm">No transactions yet.</div>'; return; }
  rct.innerHTML=`<div style="display:flex;flex-direction:column;gap:6px">${recent.map(t=>`<div class="flex-between" style="padding:7px 10px;border-radius:6px;background:var(--surface2)"><div><div style="font-size:13px;font-weight:500">${t.desc}</div><div class="text-xs text-muted">${t.date} · ${accName(t.account)}</div></div><span class="${t.type==='income'?'amount-in':t.type==='expense'?'amount-out':'amount-neutral'}">${t.type==='income'?'+':t.type==='expense'?'−':'⇄'}${fmt(t.amount)}</span></div>`).join('')}</div>`;

  // Accounts
  const ag=document.getElementById('dashAccounts');
  ag.innerHTML=(G().accounts||[]).map(a=>accountCardHTML(a,false)).join('');
}

// ══ TRANSACTION TABLE ══
function txnRow(t, showActions=true) {
  const toAcc=t.toAccount?accName(t.toAccount):'';
  const typeColor=t.type==='income'?'badge-green':t.type==='expense'?'badge-red':'badge-blue';
  const amtClass=t.type==='income'?'amount-in':t.type==='expense'?'amount-out':'amount-neutral';
  const sign=t.type==='income'?'+':t.type==='expense'?'−':'⇄';
  const actionBtns=showActions&&isAdmin()
    ?`<div class="row-actions"><button class="btn btn-outline btn-icon btn-sm" onclick="editTxn('${t.id}')" title="Edit">✏️</button><button class="btn btn-red btn-icon btn-sm" onclick="askDelete('${t.id}','txn')" title="Delete">🗑️</button></div>`
    :'';
  return `<tr><td class="font-mono text-muted" style="font-size:11px">${t.date}</td><td><div style="font-weight:500;color:var(--text)">${t.desc}</div>${t.note?`<div class="text-xs text-muted">${t.note}</div>`:''}</td><td><span class="badge badge-gray">${t.category}</span></td><td class="text-muted" style="font-size:12px">${accName(t.account)}${toAcc?' → '+toAcc:''}</td><td><span class="${amtClass}">${sign}${fmt(t.amount)}</span></td><td><span class="badge ${typeColor}">${t.type}</span></td><td style="text-align:right">${actionBtns}</td></tr>`;
}

function renderTransactions() {
  document.getElementById('txnViewBanner').innerHTML=viewOnlyBanner();
  populateCatFilter();
  const search=document.getElementById('txn-search')?.value.toLowerCase()||'';
  const tf=document.getElementById('txn-type-filter')?.value||'';
  const cf=document.getElementById('txn-cat-filter')?.value||'';
  let txns=[...(G().transactions||[])].sort((a,b)=>new Date(b.date)-new Date(a.date));
  if(tf) txns=txns.filter(t=>t.type===tf);
  if(cf) txns=txns.filter(t=>t.category===cf);
  if(search) txns=txns.filter(t=>(t.desc||'').toLowerCase().includes(search)||(t.category||'').toLowerCase().includes(search));
  const tbody=document.getElementById('txnBody');
  const empty=document.getElementById('txnEmpty');
  if(txns.length===0){ tbody.innerHTML=''; empty.style.display='block'; return; }
  empty.style.display='none'; tbody.innerHTML=txns.map(t=>txnRow(t)).join('');
}

function filterTxns() { renderTransactions(); }
function populateCatFilter() {
  const cf=document.getElementById('txn-cat-filter'); if(!cf) return;
  const cur=cf.value;
  cf.innerHTML='<option value="">All Categories</option>'+[...INCOME_CATS,...EXPENSE_CATS].map(c=>`<option value="${c}" ${c===cur?'selected':''}>${c}</option>`).join('');
}

function renderTypeTable(type) {
  const txns=[...(G().transactions||[])].filter(t=>t.type===type).sort((a,b)=>new Date(b.date)-new Date(a.date));
  const tb=document.getElementById(type==='income'?'incBody':'expBody');
  const em=document.getElementById(type==='income'?'incEmpty':'expEmpty');
  if(txns.length===0){ tb.innerHTML=''; em.style.display='block'; return; }
  em.style.display='none';
  const amtClass=type==='income'?'amount-in':'amount-out';
  const sign=type==='income'?'+':'−';
  tb.innerHTML=txns.map(t=>`<tr><td class="font-mono text-muted" style="font-size:11px">${t.date}</td><td style="font-weight:500;color:var(--text)">${t.desc}</td><td><span class="badge badge-gray">${t.category}</span></td><td class="text-muted" style="font-size:12px">${accName(t.account)}</td><td><span class="${amtClass}">${sign}${fmt(t.amount)}</span></td><td style="text-align:right">${isAdmin()?`<div class="row-actions"><button class="btn btn-outline btn-icon btn-sm" onclick="editTxn('${t.id}')">✏️</button><button class="btn btn-red btn-icon btn-sm" onclick="askDelete('${t.id}','txn')">🗑️</button></div>`:''}</td></tr>`).join('');
}

// ══ QUICK ENTRY ══
function openQuickEntry(type) {
  if(!isAdmin()){ showToast('👁️ View-only. You cannot add transactions.','error'); return; }
  navigate('transactions');
  const s=document.getElementById('quickEntrySection');
  s.style.display='block';
  setTimeout(()=>s.scrollIntoView({behavior:'smooth'}),50);
  setQType(type||'income');
  document.getElementById('q-date').value=today();
  document.getElementById('q-date-t').value=today();
}
function closeQuickEntry() { document.getElementById('quickEntrySection').style.display='none'; }
function setQType(type) {
  currentQType=type;
  ['income','expense','transfer'].forEach(t=>document.getElementById('qtype-'+t).classList.remove('active'));
  document.getElementById('qtype-'+type).classList.add('active');
  populateQSelects();
  document.getElementById('q-transfer-fields').style.display=type==='transfer'?'block':'none';
  document.getElementById('q-normal-fields').style.display=type!=='transfer'?'block':'none';
}
function populateQSelects() {
  const cats=currentQType==='income'?INCOME_CATS:currentQType==='expense'?EXPENSE_CATS:['Transfer'];
  document.getElementById('q-category').innerHTML=cats.map(c=>`<option>${c}</option>`).join('');
  const accOpts=(G().accounts||[]).map(a=>`<option value="${a.id}">${a.name}</option>`).join('');
  document.getElementById('q-account').innerHTML='<option value="">Select…</option>'+accOpts;
  document.getElementById('q-to-account').innerHTML='<option value="">Select…</option>'+accOpts;
}

async function saveQuickEntry() {
  if(!isAdmin()){ showToast('View-only','error'); return; }
  const btn=document.getElementById('q-save-btn');
  const type=currentQType;
  const desc=document.getElementById('q-desc').value.trim();
  const amount=parseFloat(document.getElementById('q-amount').value);
  const category=document.getElementById('q-category').value;
  const account=document.getElementById('q-account').value;
  const date=type==='transfer'?document.getElementById('q-date-t').value:document.getElementById('q-date').value;
  const note=type==='transfer'?document.getElementById('q-note-t').value:document.getElementById('q-note').value;
  const toAccount=document.getElementById('q-to-account').value;
  if(!desc) return showToast('⚠️ Enter a description','error');
  if(!amount||amount<=0) return showToast('⚠️ Enter a valid amount','error');
  if(!account) return showToast('⚠️ Select an account','error');
  if(!date) return showToast('⚠️ Select a date','error');
  if(type==='transfer'&&!toAccount) return showToast('⚠️ Select destination account','error');
  if(type==='transfer'&&account===toAccount) return showToast('⚠️ Source and destination must differ','error');

  // Budget check
  if(type==='expense') {
    const budget=(G().budgets||[]).find(b=>b.category===category);
    if(budget&&budget.overdraft==='no') {
      const spent=categorySpend(category);
      if(spent+amount>budget.limit) return showToast(`🚫 Budget limit for "${category}" exceeded. Blocked.`,'error');
    }
  }

  btn.disabled=true; btn.textContent='Saving…';
  try {
    const data={type,desc,category:category||'Other',account,amount,date,note:note||''};
    if(type==='transfer') data.toAccount=toAccount;
    // Update account balance
    await updateAccountBalance(type, account, amount, toAccount);
    await window.addTransaction(data);
    showToast('✓ Entry saved!','success');
    document.getElementById('q-desc').value='';
    document.getElementById('q-amount').value='';
    document.getElementById('q-note').value='';
    document.getElementById('q-note-t').value='';
    // Budget alert after save
    if(type==='expense') checkBudgetToast(category);
  } catch(e) { showToast('Error: '+e.message,'error'); }
  btn.disabled=false; btn.textContent='✓ Save Entry';
}

async function updateAccountBalance(type, accId, amount, toAccId, reverse=false) {
  const acc=(G().accounts||[]).find(a=>a.id===accId);
  if(!acc) return;
  let delta=Number(amount);
  if(type==='income') { if(reverse) delta=-delta; await window.updateAccount(accId,{balance:Number(acc.balance)+delta}); }
  else if(type==='expense') { if(!reverse) delta=-delta; await window.updateAccount(accId,{balance:Number(acc.balance)+delta}); }
  else if(type==='transfer') {
    const d=reverse?delta:-delta;
    await window.updateAccount(accId,{balance:Number(acc.balance)+d});
    const toAcc=(G().accounts||[]).find(a=>a.id===toAccId);
    if(toAcc) await window.updateAccount(toAccId,{balance:Number(toAcc.balance)-d});
  }
}

// ══ EDIT TRANSACTION ══
function editTxn(id) {
  if(!isAdmin()){ showToast('View-only','error'); return; }
  const t=(G().transactions||[]).find(x=>x.id===id);
  if(!t) return;
  document.getElementById('et-id').value=id;
  document.getElementById('et-type').value=t.type;
  document.getElementById('et-date').value=t.date;
  document.getElementById('et-desc').value=t.desc;
  document.getElementById('et-amount').value=t.amount;
  document.getElementById('et-note').value=t.note||'';
  updateEditCats();
  setTimeout(()=>{
    document.getElementById('et-category').value=t.category;
    const accOpts=(G().accounts||[]).map(a=>`<option value="${a.id}" ${a.id===t.account?'selected':''}>${a.name}</option>`).join('');
    document.getElementById('et-account').innerHTML=accOpts;
    document.getElementById('et-to-account').innerHTML=(G().accounts||[]).map(a=>`<option value="${a.id}" ${a.id===t.toAccount?'selected':''}>${a.name}</option>`).join('');
    document.getElementById('et-to-row').style.display=t.type==='transfer'?'grid':'none';
  },10);
  openModal('editTxnModal');
}
function updateEditCats() {
  const type=document.getElementById('et-type').value;
  const cats=type==='income'?INCOME_CATS:type==='expense'?EXPENSE_CATS:['Transfer'];
  document.getElementById('et-category').innerHTML=cats.map(c=>`<option>${c}</option>`).join('');
  document.getElementById('et-to-row').style.display=type==='transfer'?'grid':'none';
}
async function saveEditTxn() {
  const id=document.getElementById('et-id').value;
  const old=(G().transactions||[]).find(t=>t.id===id);
  if(!old) return;
  const updated={
    type:document.getElementById('et-type').value,
    date:document.getElementById('et-date').value,
    desc:document.getElementById('et-desc').value.trim(),
    amount:parseFloat(document.getElementById('et-amount').value),
    category:document.getElementById('et-category').value,
    account:document.getElementById('et-account').value,
    toAccount:document.getElementById('et-to-account').value||null,
    note:document.getElementById('et-note').value
  };
  if(!updated.desc||!updated.amount||!updated.account) return showToast('⚠️ Fill required fields','error');
  try {
    // Reverse old balance effect then apply new
    await updateAccountBalance(old.type,old.account,old.amount,old.toAccount,true);
    await updateAccountBalance(updated.type,updated.account,updated.amount,updated.toAccount,false);
    await window.updateTransaction(id,updated);
    closeModal('editTxnModal'); showToast('✓ Transaction updated!','success');
  } catch(e) { showToast('Error: '+e.message,'error'); }
}

// ══ DELETE ══
function askDelete(id, type) {
  if(!isAdmin()){ showToast('View-only','error'); return; }
  deleteTarget=id; deleteType=type;
  document.getElementById('confirmMsg').textContent = type==='account'?'Delete this account? Transactions will remain.':'This transaction will be permanently deleted.';
  openModal('confirmModal');
}
async function doConfirmDelete() {
  try {
    if(deleteType==='txn') {
      const t=(G().transactions||[]).find(x=>x.id===deleteTarget);
      if(t) await updateAccountBalance(t.type,t.account,t.amount,t.toAccount,true);
      await window.deleteTransaction(deleteTarget);
      showToast('🗑️ Transaction deleted','info');
    } else if(deleteType==='account') {
      await window.deleteAccount(deleteTarget); showToast('🗑️ Account deleted','info');
    } else if(deleteType==='budget') {
      await window.deleteBudgetDoc(deleteTarget); showToast('🗑️ Budget removed','info');
    } else if(deleteType==='loan') {
      await window.deleteLoanDoc(deleteTarget); showToast('🗑️ Loan removed','info');
    }
    closeModal('confirmModal');
  } catch(e) { showToast('Error: '+e.message,'error'); }
}

// ══ BUDGET ══
function getBudgetAlerts() {
  return (G().budgets||[]).map(b=>{
    const spent=categorySpend(b.category),pct=(spent/b.limit)*100,over=spent>b.limit;
    return { cat:b.category, type:over?'overdraft':'normal', pct, warnAt:b.warnAt, msg: over?`Overdraft! ${fmt(spent)} spent vs ${fmt(b.limit)} limit (${Math.round(pct)}%)`:`${Math.round(pct)}% used (${fmt(spent)} of ${fmt(b.limit)})` };
  });
}
function checkBudgetToast(cat) {
  const b=(G().budgets||[]).find(x=>x.category===cat); if(!b) return;
  const spent=categorySpend(cat),pct=(spent/b.limit)*100;
  if(spent>b.limit) showToast(`🚨 ${cat} is over budget! (${Math.round(pct)}%)`,'error');
  else if(pct>=b.warnAt) showToast(`⚠️ ${cat} at ${Math.round(pct)}% of budget`,'info');
}
function renderBudget() {
  const alerts=getBudgetAlerts();
  document.getElementById('budgetAlerts').innerHTML=alerts.filter(a=>a.type==='overdraft'||a.pct>=a.warnAt).map(a=>`<div class="overdraft-banner">⚠️ <strong>${a.cat}:</strong> ${a.msg}</div>`).join('');
  const grid=document.getElementById('budgetGrid');
  if(!G().budgets||G().budgets.length===0){ grid.innerHTML='<div class="empty-state" style="grid-column:1/-1"><div class="icon">🎯</div><p>No budgets set.</p></div>'; return; }
  grid.innerHTML=(G().budgets||[]).map(b=>{
    const spent=categorySpend(b.category),pct=(spent/b.limit)*100,over=spent>b.limit,warn=pct>=b.warnAt&&!over;
    const adminBtns=isAdmin()?`<div style="display:flex;gap:6px"><button class="btn btn-outline btn-icon btn-sm" onclick="editBudget('${b.id}')">✏️</button><button class="btn btn-red btn-icon btn-sm" onclick="askDelete('${b.id}','budget')">🗑️</button></div>`:'';
    return `<div class="budget-card"><div class="flex-between mb-2"><div><div style="font-size:13px;font-weight:600">${b.category}</div><div class="text-xs text-muted">Warn ${b.warnAt}% · ${b.overdraft==='yes'?'Overdraft OK':'Hard stop'}</div></div>${adminBtns}</div><div class="flex-between" style="font-size:11px;color:var(--text3);margin-bottom:7px"><span>Spent: <strong class="font-mono ${over?'text-red':''}">${fmt(spent)}</strong></span><span>Limit: <strong class="font-mono">${fmt(b.limit)}</strong></span><span class="font-mono ${over?'text-red':warn?'text-gold':'text-green'}">${Math.round(pct)}%</span></div><div class="progress-wrap"><div class="progress-bar ${over?'red':warn?'orange':'green'}" style="width:${Math.min(pct,100)}%"></div></div>${over?`<div class="budget-alert danger">🚨 Over budget by ${fmt(spent-b.limit)}</div>`:warn?`<div class="budget-alert warning">⚠️ ${Math.round(pct)}% used — nearing limit</div>`:''}</div>`;
  }).join('');
}
function openAddBudget() {
  if(!isAdmin()){ showToast('View-only','error'); return; }
  document.getElementById('budModalTitle').textContent='🎯 Set Budget Limit';
  document.getElementById('bud-edit-id').value='';
  const existing=(G().budgets||[]).map(b=>b.category);
  const cats=EXPENSE_CATS.filter(c=>!existing.includes(c));
  document.getElementById('bud-cat').innerHTML=cats.map(c=>`<option>${c}</option>`).join('');
  document.getElementById('bud-limit').value='';
  document.getElementById('bud-warn').value='80';
  document.getElementById('bud-overdraft').value='no';
  openModal('addBudgetModal');
}
function editBudget(id) {
  const b=(G().budgets||[]).find(x=>x.id===id); if(!b) return;
  document.getElementById('budModalTitle').textContent='✏️ Edit Budget';
  document.getElementById('bud-edit-id').value=id;
  document.getElementById('bud-cat').innerHTML=`<option>${b.category}</option>`;
  document.getElementById('bud-limit').value=b.limit;
  document.getElementById('bud-warn').value=b.warnAt;
  document.getElementById('bud-overdraft').value=b.overdraft;
  openModal('addBudgetModal');
}
async function saveBudget() {
  const cat=document.getElementById('bud-cat').value;
  const limit=parseFloat(document.getElementById('bud-limit').value);
  const warnAt=parseInt(document.getElementById('bud-warn').value)||80;
  const overdraft=document.getElementById('bud-overdraft').value;
  if(!cat||!limit) return showToast('⚠️ Fill all fields','error');
  const eid=document.getElementById('bud-edit-id').value;
  const ok=await window.saveBudgetDoc(eid||null,{category:cat,limit,warnAt,overdraft});
  if(ok){ closeModal('addBudgetModal'); showToast('✓ Budget saved!','success'); }
}

// ══ ACCOUNTS ══
function accountCardHTML(a, showActions=true) {
  const col=COLORS[a.color]||COLORS.blue;
  const bal=Number(a.balance||0);
  const btns=showActions&&isAdmin()?`<div style="display:flex;gap:6px;margin-top:12px"><button class="btn btn-outline btn-sm" onclick="event.stopPropagation();editAccount('${a.id}')">✏️ Edit</button><button class="btn btn-red btn-sm" onclick="event.stopPropagation();askDelete('${a.id}','account')">🗑️</button></div>`:'';
  return `<div class="account-card" onclick="focusAccTab('${a.id}')"><div class="account-deco" style="background:${col}"></div><div class="account-type-badge" style="color:${col}">${ACC_TYPE_LABELS[a.type]||a.type}</div><div class="account-name">${a.name}</div>${a.note?`<div class="text-xs text-muted mt-1">${a.note}</div>`:''}<div class="account-balance" style="color:${col}">${fmt(bal)}</div>${btns}</div>`;
}
function renderAccounts() {
  const grid=document.getElementById('accountsGrid');
  if(!G().accounts||G().accounts.length===0){ grid.innerHTML='<div class="empty-state" style="grid-column:1/-1"><div class="icon">🏦</div><p>No accounts added.</p></div>'; return; }
  grid.innerHTML=(G().accounts||[]).map(a=>accountCardHTML(a)).join('');
  const tabs=document.getElementById('accountTabs');
  if(G().accounts.length>0){
    if(!activeAccTab||!(G().accounts||[]).find(a=>a.id===activeAccTab)) activeAccTab=(G().accounts[0]||{}).id;
    tabs.innerHTML=(G().accounts||[]).map(a=>`<div class="tab ${a.id===activeAccTab?'active':''}" onclick="focusAccTab('${a.id}')">${a.name}</div>`).join('');
    renderAccTxns(activeAccTab);
  }
}
function focusAccTab(id) {
  activeAccTab=id;
  document.querySelectorAll('#accountTabs .tab').forEach(t=>t.classList.remove('active'));
  document.querySelectorAll('#accountTabs .tab').forEach(t=>{ if(t.getAttribute('onclick')?.includes(id)) t.classList.add('active'); });
  renderAccTxns(id);
}
function renderAccTxns(accId) {
  const txns=(G().transactions||[]).filter(t=>t.account===accId||t.toAccount===accId).sort((a,b)=>new Date(b.date)-new Date(a.date));
  const tb=document.getElementById('accTxnBody');
  const em=document.getElementById('accTxnEmpty');
  if(txns.length===0){ tb.innerHTML=''; em.style.display='block'; return; }
  em.style.display='none';
  tb.innerHTML=txns.map(t=>{
    const isTo=t.toAccount===accId&&t.type==='transfer';
    const amtClass=isTo?'amount-in':t.type==='income'?'amount-in':t.type==='expense'?'amount-out':'amount-neutral';
    const sign=isTo?'+':t.type==='income'?'+':t.type==='expense'?'−':'⇄';
    const typeColor=t.type==='income'?'badge-green':t.type==='expense'?'badge-red':'badge-blue';
    return `<tr><td class="font-mono text-muted" style="font-size:11px">${t.date}</td><td style="font-weight:500;color:var(--text)">${t.desc}</td><td><span class="badge badge-gray">${t.category}</span></td><td><span class="${amtClass}">${sign}${fmt(t.amount)}</span></td><td><span class="badge ${typeColor}">${t.type}</span></td><td style="text-align:right">${isAdmin()?`<div class="row-actions"><button class="btn btn-outline btn-icon btn-sm" onclick="editTxn('${t.id}')">✏️</button><button class="btn btn-red btn-icon btn-sm" onclick="askDelete('${t.id}','txn')">🗑️</button></div>`:''}</td></tr>`;
  }).join('');
}
function openAddAccount() {
  if(!isAdmin()){ showToast('View-only','error'); return; }
  document.getElementById('accModalTitle').textContent='🏦 Add Account';
  document.getElementById('acc-edit-id').value='';
  document.getElementById('acc-name').value='';
  document.getElementById('acc-type').value='savings';
  document.getElementById('acc-balance').value='0';
  document.getElementById('acc-color').value='blue';
  document.getElementById('acc-note').value='';
  openModal('addAccountModal');
}
function editAccount(id) {
  const a=(G().accounts||[]).find(x=>x.id===id); if(!a) return;
  document.getElementById('accModalTitle').textContent='✏️ Edit Account';
  document.getElementById('acc-edit-id').value=id;
  document.getElementById('acc-name').value=a.name;
  document.getElementById('acc-type').value=a.type;
  document.getElementById('acc-balance').value=a.balance;
  document.getElementById('acc-color').value=a.color||'blue';
  document.getElementById('acc-note').value=a.note||'';
  openModal('addAccountModal');
}
async function saveAccount() {
  const name=document.getElementById('acc-name').value.trim();
  const type=document.getElementById('acc-type').value;
  const balance=parseFloat(document.getElementById('acc-balance').value)||0;
  const color=document.getElementById('acc-color').value;
  const note=document.getElementById('acc-note').value.trim();
  if(!name) return showToast('⚠️ Enter account name','error');
  const eid=document.getElementById('acc-edit-id').value;
  try {
    if(eid) await window.updateAccount(eid,{name,type,balance,color,note});
    else await window.addAccount({name,type,balance,color,note});
    closeModal('addAccountModal'); showToast('✓ Account saved!','success');
  } catch(e) { showToast('Error: '+e.message,'error'); }
}

// ══ LOANS ══
function renderLoans() {
  const grid=document.getElementById('loansGrid');
  if(!G().loans||G().loans.length===0){ grid.innerHTML='<div class="empty-state" style="grid-column:1/-1"><div class="icon">💳</div><p>No loans added.</p></div>'; return; }
  grid.innerHTML=(G().loans||[]).map(l=>`<div class="card"><div class="flex-between mb-2"><div><div style="font-size:15px;font-weight:600">${l.name}</div><div class="text-xs text-muted">${l.lender||''}</div></div>${isAdmin()?`<button class="btn btn-red btn-icon btn-sm" onclick="askDelete('${l.id}','loan')">🗑️</button>`:''}</div><div style="display:grid;grid-template-columns:1fr 1fr 1fr;gap:10px;margin-bottom:12px"><div><div class="text-xs text-muted">Principal</div><div class="font-mono">${fmt(l.principal)}</div></div><div><div class="text-xs text-muted">Rate</div><div class="font-mono">${l.rate}%</div></div><div><div class="text-xs text-muted">Tenure</div><div class="font-mono">${l.tenure} mo</div></div></div><div class="flex-between" style="background:var(--surface2);border-radius:8px;padding:10px 14px"><span class="text-sm">Monthly EMI</span><span class="font-mono text-red" style="font-size:16px">${fmt(l.emi)}</span></div>${l.due?`<div class="text-xs text-muted mt-1">Next due: ${l.due}</div>`:''}</div>`).join('');
}
async function saveLoan() {
  const name=document.getElementById('loan-name').value.trim();
  const principal=parseFloat(document.getElementById('loan-principal').value)||0;
  if(!name||!principal) return showToast('⚠️ Fill required fields','error');
  await window.saveLoanDoc({name,lender:document.getElementById('loan-lender').value,principal,rate:parseFloat(document.getElementById('loan-rate').value)||0,tenure:parseInt(document.getElementById('loan-tenure').value)||0,emi:parseFloat(document.getElementById('loan-emi').value)||0,due:document.getElementById('loan-due').value});
  closeModal('addLoanModal'); showToast('✓ Loan added!','success');
}

// ══ USERS (ADMIN) ══
function renderUsers() {
  const users=G().allUsers||[];
  const tb=document.getElementById('usersTable');
  if(users.length===0){ tb.innerHTML='<div class="empty-state"><div class="icon">👥</div><p>No users yet.</p></div>'; return; }
  tb.innerHTML=users.map(u=>{
    const isMe=u.id===G().user.uid;
    return `<div class="user-row"><div><div style="font-weight:600;font-size:13px">${u.name||'—'}</div><div class="text-xs text-muted">${u.email||''}</div></div><div><span class="badge ${u.role==='admin'?'badge-green':'badge-blue'}">${u.role||'viewer'}</span></div><div><select onchange="doUpdateUserRole('${u.id}',this.value)" ${isMe?'disabled':''} style="background:var(--surface2);border:1px solid var(--border);border-radius:6px;color:var(--text);padding:5px 8px;font-size:12px;outline:none"><option value="viewer" ${u.role==='viewer'?'selected':''}>Viewer</option><option value="admin" ${u.role==='admin'?'selected':''}>Admin</option></select></div><div>${!isMe?`<button class="btn btn-red btn-sm" onclick="doDeleteUser('${u.id}')">Remove</button>`:'<span class="text-xs text-muted">You</span>'}</div></div>`;
  }).join('');
}

// ══ SUMMARY ══
function renderSummary() {
  const inc=monthlyIncome(),exp=monthlyExpense(),sav=inc-exp;
  document.getElementById('summaryStats').innerHTML=`<div class="stat-card green"><div class="stat-icon">💵</div><div class="stat-label">Income</div><div class="stat-value green">${fmt(inc)}</div></div><div class="stat-card red"><div class="stat-icon">🧾</div><div class="stat-label">Expenses</div><div class="stat-value red">${fmt(exp)}</div></div><div class="stat-card gold"><div class="stat-icon">💎</div><div class="stat-label">Net Savings</div><div class="stat-value ${sav>=0?'gold':'red'}">${fmt(sav)}</div></div><div class="stat-card blue"><div class="stat-icon">🏦</div><div class="stat-label">Total Assets</div><div class="stat-value blue">${fmt(totalAssets())}</div></div>`;
  const byCat=(type)=>{ const m={};thisMonthTxns().filter(t=>t.type===type).forEach(t=>{m[t.category]=(m[t.category]||0)+Number(t.amount)});return Object.entries(m).sort((a,b)=>b[1]-a[1]); };
  const renderBk=(id,entries,cls)=>{ const total=entries.reduce((s,e)=>s+e[1],0); if(!entries.length){document.getElementById(id).innerHTML='<div class="text-muted text-sm">No data</div>';return;} document.getElementById(id).innerHTML=entries.map(([cat,amt])=>`<div style="margin-bottom:10px"><div class="flex-between mb-1"><span style="font-size:13px">${cat}</span><span class="font-mono text-sm ${cls}">${fmt(amt)}</span></div><div class="progress-wrap"><div class="progress-bar ${cls.includes('green')?'green':'red'}" style="width:${total?Math.round((amt/total)*100):0}%"></div></div></div>`).join(''); };
  renderBk('summaryInc',byCat('income'),'text-green');
  renderBk('summaryExp',byCat('expense'),'text-red');
  const txns=thisMonthTxns().sort((a,b)=>new Date(b.date)-new Date(a.date));
  document.getElementById('summaryBody').innerHTML=txns.map(t=>txnRow(t,false)).join('')||'<tr><td colspan="6" class="text-muted" style="text-align:center;padding:24px">No transactions this month</td></tr>';
}

// ══ MODAL HELPERS ══
function openModal(id) { document.getElementById(id).classList.add('open'); }
function closeModal(id) { document.getElementById(id).classList.remove('open'); }
document.querySelectorAll('.modal-overlay').forEach(m=>{ m.addEventListener('click',e=>{ if(e.target===m) closeModal(m.id); }); });

// ══ TOAST ══
let toastTimer;
function showToast(msg, type='success') {
  const el=document.getElementById('toast');
  el.textContent=msg; el.className=`toast ${type} show`;
  clearTimeout(toastTimer);
  toastTimer=setTimeout(()=>el.classList.remove('show'),3500);
}
// Make it accessible from module scope too
window.showToast=showToast;
</script>
</body>
</html>
