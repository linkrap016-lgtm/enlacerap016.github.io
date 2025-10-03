<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>App de Publicación con Google Auth</title>
    <!-- Carga de Tailwind CSS para estilos -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Configuración de fuente y estilos base */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        .card {
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
        }
    </style>
</head>
<body>
    <div class="min-h-screen flex items-center justify-center p-4">
        <div class="card bg-white p-8 rounded-xl w-full max-w-md text-center">
            <h1 class="text-3xl font-extrabold text-gray-900 mb-6">Plataforma de Publicación</h1>
            
            <!-- Contenedor del Estado de Autenticación -->
            <div id="authStatus" class="mb-6 p-4 rounded-lg bg-indigo-50 border border-indigo-200">
                <p class="text-indigo-700 font-medium">Cargando estado...</p>
            </div>

            <!-- Botón de Acción Principal (Login/Logout) -->
            <button id="authButton" 
                    class="w-full py-3 px-4 mb-4 text-white font-semibold rounded-lg transition duration-300 ease-in-out transform hover:scale-[1.02] active:scale-[0.98] focus:outline-none focus:ring-4 focus:ring-indigo-300"
                    disabled>
                Cargando...
            </button>

            <!-- Botón de Publicar (Solo visible para usuarios autenticados) -->
            <button id="publishButton" 
                    onclick="publishPost()" 
                    class="w-full py-3 px-4 bg-green-600 text-white font-semibold rounded-lg transition duration-300 ease-in-out transform hover:bg-green-700 hover:scale-[1.02] active:scale-[0.98] focus:outline-none focus:ring-4 focus:ring-green-300 hidden"
                    disabled>
                Publicar Contenido de Prueba
            </button>
            
            <!-- Área de Mensajes -->
            <div id="messageBox" class="mt-4 p-3 text-sm rounded-lg transition-opacity duration-500 opacity-0"></div>
        </div>
    </div>

    <!-- Firebase SDKs -->
    <script type="module">
        // Importaciones necesarias de Firebase
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        // Se añade signInWithPopup para la funcionalidad solicitada
        import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged, GoogleAuthProvider, signOut, signInWithPopup } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        // Se asegura el uso de addDoc
        import { getFirestore, collection, addDoc, doc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 1. Configuración y Inicialización de Firebase
        // Variables globales proporcionadas por el entorno Canvas
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        
        // Inicializar Firebase
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);

        let currentUserId = null;

        // Elementos del DOM
        const authStatusEl = document.getElementById('authStatus');
        const authButton = document.getElementById('authButton');
        const publishButton = document.getElementById('publishButton');
        const messageBox = document.getElementById('messageBox');

        // Función auxiliar para mostrar mensajes
        function showMessage(text, isError = false) {
            messageBox.textContent = text;
            messageBox.className = `mt-4 p-3 text-sm rounded-lg transition-opacity duration-500 opacity-100 ${isError ? 'bg-red-100 text-red-700' : 'bg-green-100 text-green-700'}`;
            
            // Ocultar mensaje después de 5 segundos
            setTimeout(() => {
                messageBox.classList.add('opacity-0');
            }, 5000);
        }

        // 2. Autenticación Inicial y Escucha de Cambios de Estado
        if (initialAuthToken) {
            // Intento de inicio de sesión con el token de Firebase proporcionado por el entorno
            signInWithCustomToken(auth, initialAuthToken).catch(error => {
                console.error("Error al iniciar sesión con token personalizado:", error);
                signInAnonymously(auth);
            });
        } else {
            // Si no hay token, inicia sesión de forma anónima
            signInAnonymously(auth);
        }

        // Listener de cambios en el estado de autenticación
        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUserId = user.uid;
                
                // Determinamos si es un usuario completo (no anónimo). 
                const isFullUser = !user.isAnonymous; 
                
                let displayName = user.displayName || 'Usuario de Plataforma';
                let actionText = 'Cerrar Sesión';
                let authClass = 'bg-red-500 hover:bg-red-600 focus:ring-red-300';
                
                if (!isFullUser) { // Es anónimo
                     displayName = 'No has iniciado sesión (Anónimo)';
                     actionText = 'Iniciar Sesión con Google';
                     authClass = 'bg-indigo-600 hover:bg-indigo-700 focus:ring-indigo-300';
                     publishButton.classList.add('hidden');
                     publishButton.disabled = true;
                } else {
                    // Es un usuario completo (logueado con el token inicial)
                    authStatusEl.innerHTML = `<p class="text-indigo-700 font-medium">Sesión Iniciada como:</p><p class="text-lg font-bold truncate">${displayName}</p><p class="text-xs text-gray-500 mt-1">ID: ${currentUserId}</p>`;
                    publishButton.classList.remove('hidden');
                    publishButton.disabled = false;
                }
                
                authButton.textContent = actionText;
                // Reemplazamos las clases de color de forma segura
                authButton.className = authButton.className.replace(/bg-.*|hover:bg-.*|focus:ring-.*/g, '').trim() + ' ' + authClass;
                authButton.onclick = isFullUser ? signOutUser : signInWithGoogle;

            } else {
                // Usuario desconectado
                currentUserId = null;
                authStatusEl.innerHTML = `<p class="text-red-700 font-medium">Sesión Cerrada.</p><p class="text-sm text-red-500">Inicia sesión para publicar.</p>`;
                authButton.textContent = 'Iniciar Sesión con Google';
                authButton.className = authButton.className.replace(/bg-.*|hover:bg-.*|focus:ring-.*/g, '').trim() + ' bg-indigo-600 hover:bg-indigo-700 focus:ring-indigo-300';
                authButton.onclick = signInWithGoogle;
                publishButton.classList.add('hidden');
                publishButton.disabled = true;
            }
            authButton.disabled = false; // Habilitar el botón una vez que el estado es claro
        });

        // 3. Funciones de Autenticación
        window.signInWithGoogle = async () => {
            // NOTA IMPORTANTE: Esta función intenta usar el método estándar de Google Sign-In 
            // mediante ventana emergente, tal como fue solicitado.
            // Es probable que este método FALLE en el entorno de Canvas/iFrame con el error: 
            // 'auth/unauthorized-domain', ya que el dominio no está autorizado para pop-ups.
            
            const provider = new GoogleAuthProvider();
            try {
                // Intento de iniciar sesión con ventana emergente
                await signInWithPopup(auth, provider);
                showMessage('¡Inicio de sesión con Google exitoso!', false);
            } catch (error) {
                // Cambiamos el error a console.warn ya que es un comportamiento esperado en este entorno
                console.warn("Error al iniciar sesión con Google (auth/unauthorized-domain esperado en iFrame). Aplicando fallback:", error);
                
                // --- Lógica de Respaldo (Fallback) ---
                // Si falla el pop-up (que es lo esperado en este entorno), 
                // intentamos el método de la plataforma como fallback para que el usuario pueda publicar.
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                    showMessage('⚠️ Falló la ventana emergente. Sesión iniciada usando el token de plataforma.', true);
                } else {
                    showMessage(`Error: ${error.message}. No se pudo iniciar sesión.`, true);
                }
            }
        };

        window.signOutUser = async () => {
            try {
                await signOut(auth);
                // El listener onAuthStateChanged maneja el cambio a estado Anónimo/Desconectado
                showMessage('Has cerrado sesión correctamente.', false);
            } catch (error) {
                 console.error("Error al cerrar sesión:", error);
                 showMessage(`Error al cerrar sesión: ${error.message}`, true);
            }
        };


        // 4. Función de Publicación y Guardado en Firestore
        window.publishPost = async () => {
            if (!currentUserId || auth.currentUser.isAnonymous) {
                showMessage('Debes iniciar sesión con Google para publicar.', true);
                return;
            }

            // Datos de la publicación de prueba
            const postData = {
                titulo: "Publicación de Prueba " + new Date().toLocaleTimeString(),
                contenido: "Este es el contenido de prueba. Se ha guardado usando el UID: " + currentUserId,
                fecha: new Date(),
                autorId: currentUserId
            };

            try {
                // Ruta a la colección privada del usuario: /artifacts/{appId}/users/{userId}/publicaciones
                const postCollectionRef = collection(db, 'artifacts', appId, 'users', currentUserId, 'publicaciones');
                
                // Usamos addDoc para garantizar un nuevo ID único cada vez, solucionando el error 'Document already exists'.
                const docRef = await addDoc(postCollectionRef, postData);
                
                showMessage(`¡Publicación exitosa! Documento guardado con ID: ${docRef.id}`);

            } catch (error) {
                console.error("Error al publicar:", error);
                showMessage(`Error al publicar contenido: ${error.message}`, true);
            }
        };
    </script>
</body>
</html>
