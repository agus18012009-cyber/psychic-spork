<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Sistema de Control - Campaña Ambiental</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, addDoc, collection, query, where, getDocs, serverTimestamp, deleteDoc, onSnapshot, orderBy } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

        // CONFIGURACIÓN DE TU PROYECTO
        const firebaseConfig = {
            apiKey: "AIzaSyDvFWH-MjkxtMtrW74GE50oR0XJ2xnYOPg",
            authDomain: "campana-papel.firebaseapp.com",
            projectId: "campana-papel",
            storageBucket: "campana-papel.firebasestorage.app",
            messagingSenderId: "422591862597",
            appId: "1:422591862597:web:54a1a8756bfa01376fd02c",
            measurementId: "G-ZHJ8FGS0N0"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        const CLAVE_ADMIN = "ADMIN123"; 
        let currentUnsubscribe = null;

        const getMesActual = () => {
            const meses = ["Enero", "Febrero", "Marzo", "Abril", "Mayo", "Junio", "Julio", "Agosto", "Septiembre", "Octubre", "Noviembre", "Diciembre"];
            return meses[new Date().getMonth()] + " " + new Date().getFullYear();
        };

        // REGISTRO
        window.register = async () => {
            const email = document.getElementById('email').value;
            const pass = document.getElementById('pass').value;
            const name = document.getElementById('fullName').value;
            const curso = document.getElementById('myCourse').value;
            const code = document.getElementById('adminCode').value;

            if(!email || !pass || !name) return alert("Por favor, completa todos los campos obligatorios.");
            let role = (code === CLAVE_ADMIN) ? "admin" : "student";

            try {
                const res = await createUserWithEmailAndPassword(auth, email, pass);
                await setDoc(doc(db, "users", res.user.uid), { name, curso, role });
                alert("Cuenta creada correctamente como: " + role.toUpperCase());
            } catch (e) { alert("Error: " + e.message); }
        };

        // LOGIN
        window.login = async () => {
            const email = document.getElementById('email').value;
            const pass = document.getElementById('pass').value;
            try { await signInWithEmailAndPassword(auth, email, pass); } 
            catch (e) { alert("Error al ingresar. Revisa tus datos."); }
        };

        // MARCAR VISITA
        window.markVisit = async () => {
            const target = document.getElementById('targetCourse').value;
            const user = auth.currentUser;
            const userData = (await getDoc(doc(db, "users", user.uid))).data();
            const mes = getMesActual();

            const q = query(collection(db, "visits"), 
                where("studentCourse", "==", userData.curso), 
                where("targetCourse", "==", target),
                where("mes", "==", mes)
            );
            const snap = await getDocs(q);
            if (!snap.empty) return alert("Este curso ya fue registrado por alguien de tu división este mes.");

            try {
                await addDoc(collection(db, "visits"), {
                    studentName: userData.name,
                    studentId: user.uid,
                    studentCourse: userData.curso,
                    targetCourse: target,
                    mes: mes,
                    timestamp: serverTimestamp()
                });
                alert("Registro guardado con éxito.");
            } catch (e) { alert("Error al guardar registro."); }
        };

        // ELIMINAR REGISTRO (SOLO ADMIN)
        window.deleteEntry = async (id) => {
            if(confirm("¿Estás seguro de que quieres eliminar este registro?")) {
                await deleteDoc(doc(db, "visits", id));
            }
        };

        // ESCUCHAR CAMBIOS EN TIEMPO REAL
        function listenToCourse(courseName, userRole) {
            if (currentUnsubscribe) currentUnsubscribe();

            const q = query(
                collection(db, "visits"), 
                where("studentCourse", "==", courseName),
                where("mes", "==", getMesActual())
            );

            currentUnsubscribe = onSnapshot(q, (snapshot) => {
                let html = "";
                snapshot.forEach(docSnap => {
                    const d = docSnap.data();
                    const id = docSnap.id;
                    
                    const f = d.timestamp?.toDate();
                    const fechaTxt = f ? f.toLocaleDateString() : "Cargando...";
                    const horaTxt = f ? f.toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'}) : "...";

                    html += `
                    <div class="bg-white p-4 rounded-xl mb-3 shadow-sm border border-gray-100 flex justify-between items-center">
                        <div>
                            <div class="flex flex-wrap items-center gap-2 mb-1">
                                <span class="bg-blue-100 text-blue-700 text-[10px] font-bold px-2 py-0.5 rounded-md uppercase">${d.targetCourse}</span>
                                <span class="text-[10px] text-gray-400 font-medium">${fechaTxt} | ${horaTxt}</span>
                            </div>
                            <p class="text-sm font-bold text-gray-800">${d.studentName}</p>
                            <p class="text-[9px] text-gray-400 font-bold uppercase tracking-widest">${d.studentCourse}</p>
                        </div>
                        ${userRole === 'admin' ? `
                            <button onclick="deleteEntry('${id}')" class="bg-red-50 p-2 rounded-lg text-red-500 active:bg-red-200">
                                🗑️
                            </button>
                        ` : ''}
                    </div>`;
                });
                document.getElementById('mainList').innerHTML = html || "<p class='text-center text-gray-400 text-sm py-10'>No hay registros en este curso.</p>";
            });
        }

        // CONTROL DE SESIÓN
        onAuthStateChanged(auth, async (user) => {
            if (user) {
                const userData = (await getDoc(doc(db, "users", user.uid))).data();
                document.getElementById('auth-ui').classList.add('hidden');
                document.getElementById('app-ui').classList.remove('hidden');
                document.getElementById('uName').innerText = userData.name;
                document.getElementById('uDiv').innerText = userData.curso;

                if(userData.role === "admin") {
                    document.getElementById('admin-tools').classList.remove('hidden');
                    // Iniciar viendo su propio curso
                    listenToCourse(userData.curso, "admin");
                    document.getElementById('viewCourseSelector').value = userData.curso;
                } else {
                    listenToCourse(userData.curso, "student");
                }

                // Cambio de curso manual para admin
                document.getElementById('viewCourseSelector').onchange = (e) => {
                    listenToCourse(e.target.value, userData.role);
                };

            } else {
                document.getElementById('auth-ui').classList.remove('hidden');
                document.getElementById('app-ui').classList.add('hidden');
            }
        });

        window.logout = () => signOut(auth);
    </script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; }
        input, select, button { font-size: 16px !important; } /* Evita zoom en iPhone */
    </style>
</head>
<body class="bg-gray-50 text-gray-900">

    <div class="max-w-md mx-auto min-h-screen bg-white shadow-xl flex flex-col">
        
        <!-- HEADER -->
        <header class="bg-blue-600 p-6 text-white pt-10">
            <h1 class="text-2xl font-black tracking-tighter uppercase italic">Control Ambiental</h1>
            <p class="text-[10px] font-bold opacity-80 tracking-widest uppercase mt-1">Campaña de Concientización</p>
        </header>

        <main class="p-5 flex-grow">
            
            <!-- REGISTRO / LOGIN -->
            <div id="auth-ui" class="space-y-4">
                <div class="bg-blue-50 p-5 rounded-3xl border border-blue-100">
                    <h2 class="text-sm font-black text-blue-800 mb-4 uppercase text-center">Identificación</h2>
                    <input type="text" id="fullName" placeholder="Nombre y Apellido completo" class="w-full p-4 rounded-2xl border-none mb-3 shadow-sm outline-none">
                    <select id="myCourse" class="w-full p-4 rounded-2xl border-none mb-3 shadow-sm outline-none bg-white font-bold">
                        <option value="Quinto Primera">Quinto Primera</option>
                        <option value="Quinto Segunda">Quinto Segunda</option>
                        <option value="Quinto Tercera">Quinto Tercera</option>
                    </select>
                    <input type="email" id="email" placeholder="Correo electrónico" class="w-full p-4 rounded-2xl border-none mb-3 shadow-sm outline-none">
                    <input type="password" id="pass" placeholder="Contraseña" class="w-full p-4 rounded-2xl border-none mb-4 shadow-sm outline-none">
                    
                    <p class="text-[10px] text-gray-400 font-bold uppercase mb-2 ml-1 text-center italic">Solo para encargados:</p>
                    <input type="text" id="adminCode" placeholder="Clave Administrador" class="w-full p-2 bg-transparent border-b border-gray-300 outline-none text-center text-sm mb-6">
                    
                    <button onclick="login()" class="w-full bg-blue-600 text-white p-4 rounded-2xl font-black shadow-lg mb-2 active:scale-95 transition-transform">INGRESAR</button>
                    <button onclick="register()" class="w-full text-blue-600 font-bold text-xs py-2 uppercase">Registrar cuenta nueva</button>
                </div>
            </div>

            <!-- CONTENIDO APP -->
            <div id="app-ui" class="hidden space-y-6">
                
                <!-- PERFIL -->
                <div class="flex justify-between items-center border-b pb-4">
                    <div>
                        <p id="uName" class="font-black text-gray-800 leading-none mb-1"></p>
                        <p id="uDiv" class="text-[10px] font-bold text-blue-600 uppercase tracking-widest"></p>
                    </div>
                    <button onclick="logout()" class="text-[10px] font-black text-red-400 uppercase tracking-widest border border-red-100 px-3 py-1 rounded-full">Salir</button>
                </div>

                <!-- ACCION REGISTRAR -->
                <div class="bg-slate-900 p-6 rounded-[2rem] text-white shadow-2xl">
                    <h3 class="text-xs font-black uppercase text-blue-400 mb-4 text-center tracking-widest">Registrar nueva visita</h3>
                    <select id="targetCourse" class="w-full p-4 rounded-2xl bg-slate-800 text-white border-none font-bold text-lg mb-4 outline-none appearance-none">
                        <optgroup label="1er Año" class="text-gray-400">
                            <option>1ro 1ra</option><option>1ro 2da</option><option>1ro 3ra</option><option>1ro 4ta</option><option>1ro 5ta</option><option>1ro 6ta</option>
                        </optgroup>
                        <optgroup label="2do Año" class="text-gray-400">
                            <option>2do 1ra</option><option>2do 2da</option><option>2do 3ra</option><option>2do 4ta</option><option>2do 5ta</option>
                        </optgroup>
                        <optgroup label="3er Año" class="text-gray-400">
                            <option>3ro 1ra</option><option>3ro 2da</option><option>3ro 3ra</option><option>3ro 4ta</option>
                        </optgroup>
                        <optgroup label="4to Año" class="text-gray-400">
                            <option>4to 1ra</option><option>4to 2da</option><option>4to 3ra</option><option>4to 4ta</option>
                        </optgroup>
                    </select>
                    <button onclick="markVisit()" class="w-full bg-blue-500 text-white p-5 rounded-2xl font-black text-lg shadow-xl shadow-blue-900/50 active:scale-95 transition-transform">GUARDAR AHORA</button>
                </div>

                <!-- HERRAMIENTAS DE SUPERVISIÓN (SOLO ADMIN) -->
                <div id="admin-tools" class="hidden bg-gray-100 p-4 rounded-2xl border border-gray-200">
                    <h3 class="text-[10px] font-black text-gray-500 uppercase mb-3 text-center">Supervisar otro curso</h3>
                    <select id="viewCourseSelector" class="w-full p-3 rounded-xl border-none shadow-sm font-bold text-gray-700 outline-none">
                        <option value="Quinto Primera">Quinto Primera</option>
                        <option value="Quinto Segunda">Quinto Segunda</option>
                        <option value="Quinto Tercera">Quinto Tercera</option>
                    </select>
                </div>

                <!-- LISTADO DE VISITAS -->
                <div>
                    <h3 class="text-xs font-black text-gray-400 uppercase tracking-widest mb-4 ml-1">Registros cargados:</h3>
                    <div id="mainList" class="space-y-1">
                        <!-- Los datos aparecen aquí -->
                    </div>
                </div>

            </div>
        </main>

        <footer class="p-6 text-center text-[9px] text-gray-300 font-bold uppercase tracking-[0.3em]">
            Base de datos en tiempo real
        </footer>
    </div>

</body>
</html>
