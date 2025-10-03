<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Narrativa - Tu Plataforma de Historias</title>
    <!-- Carga de Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Fuente Inter -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800&display=swap" rel="stylesheet">
    <!-- Iconos Lucide (Opcional, pero genial para UI) -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        :root {
            font-family: 'Inter', sans-serif;
        }
        /* Estilos personalizados para el scrollbar */
        .story-feed::-webkit-scrollbar {
            width: 8px;
        }
        .story-feed::-webkit-scrollbar-thumb {
            background-color: #4a5568; /* Gris oscuro para el pulgar */
            border-radius: 4px;
        }
        .story-feed::-webkit-scrollbar-track {
            background: #2d3748; /* Fondo más oscuro */
        }
        .modal {
            transition: opacity 0.3s ease-in-out, transform 0.3s ease-in-out;
        }
        /* Estilo para etiquetas seleccionadas */
        .tag-selected {
            background-color: #6b46c1; /* purple-600 */
            border-color: #a78bfa; /* purple-400 */
        }
    </style>
</head>
<body class="bg-gray-900 text-gray-100 min-h-screen">

    <!-- Navbar -->
    <nav class="bg-gray-800 shadow-xl sticky top-0 z-50">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div class="flex items-center justify-between h-16">
                <!-- Logo -->
                <div class="flex-shrink-0">
                    <span class="text-2xl font-extrabold text-purple-400">Narrativa</span>
                </div>
                <!-- Opciones de Navegación (5 Opciones) -->
                <div class="flex space-x-4 md:space-x-8">
                    <button data-page="home" class="nav-link text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium transition duration-150 ease-in-out active-link">Inicio</button>
                    <button data-page="explore" class="nav-link text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium transition duration-150 ease-in-out">Explorar</button>
                    <button data-page="create" class="nav-link text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium transition duration-150 ease-in-out">Crear Historia</button>
                    <button data-page="my-stories" class="nav-link text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium transition duration-150 ease-in-out">Mis Historias</button>
                    <button data-page="profile" class="nav-link text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium transition duration-150 ease-in-out">Perfil</button>
                </div>
            </div>
        </div>
    </nav>

    <!-- Contenido Principal -->
    <main id="main-content" class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8 md:py-12">
        <!-- El contenido de la página se inyectará aquí -->
        <div id="content-container">
            <!-- Loading State -->
            <div id="loading-spinner" class="flex justify-center items-center h-64">
                <div class="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-purple-500"></div>
                <p class="ml-4 text-purple-400 font-semibold">Cargando la base de datos de historias...</p>
            </div>
        </div>
    </main>

    <!-- Modal para Errores/Mensajes (Reemplaza a alert/confirm) -->
    <div id="app-modal" class="fixed inset-0 bg-gray-900 bg-opacity-75 hidden flex items-center justify-center p-4 z-[9999]">
        <div class="modal bg-gray-800 p-6 rounded-xl shadow-2xl max-w-sm w-full transform scale-95 transition-all">
            <h3 id="modal-title" class="text-xl font-bold text-purple-400 mb-4"></h3>
            <p id="modal-message" class="text-gray-300 mb-6"></p>
            <button id="modal-close-btn" class="w-full bg-purple-600 hover:bg-purple-700 text-white font-semibold py-2 rounded-lg transition duration-150">Aceptar</button>
        </div>
    </div>


    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, GoogleAuthProvider, signInWithPopup, updateProfile } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, serverTimestamp, setDoc, getDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase variables
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app;
        let db;
        let auth;
        let userId = null;
        let isAuthReady = false;
        let userDisplayName = 'Cargando...'; // Nombre de usuario para mostrar

        // Base de datos de etiquetas/prompts
        const TAGS = [
            'Fantasía', 'Ciencia Ficción', 'Romance', 'Misterio', 'Thriller',
            'Aventura', 'Drama', 'Comedia', 'Histórica', 'Poesía',
            'Steampunk', 'Ciberpunk', 'Magia', 'Vampiros', 'Hombres Lobo',
            'Distopía', 'Utopía', 'Viaje en el Tiempo', 'Post-Apocalíptico',
            'Fanfic', 'Terror', 'Ficción Juvenil', 'Acción', 'Filosofía'
        ];
        let selectedTags = [];
        let coverPreviewUrl = null; // URL local para la previsualización

        // Firestore Path for Public Data (all users can see/contribute)
        const STORIES_COLLECTION_PATH = `/artifacts/${appId}/public/data/stories`;
        // Firestore Path for Private User Metadata
        const getUserProfileDocRef = (uid) => doc(db, `/artifacts/${appId}/users/${uid}/metadata`, 'profile');

        // --- Utilidades UI ---
        const contentContainer = document.getElementById('content-container');
        const loadingSpinner = document.getElementById('loading-spinner');
        const navLinks = document.querySelectorAll('.nav-link');
        let currentPage = 'home'; // Estado de la navegación

        function showModal(title, message) {
            const modal = document.getElementById('app-modal');
            document.getElementById('modal-title').textContent = title;
            document.getElementById('modal-message').textContent = message;
            modal.classList.remove('hidden');
            document.getElementById('modal-close-btn').onclick = () => {
                modal.classList.add('hidden');
            };
        }

        function renderContent(html) {
            contentContainer.innerHTML = html;
            loadingSpinner.classList.add('hidden');
            lucide.createIcons();
        }

        function updateNavBar(activePage) {
            navLinks.forEach(link => {
                link.classList.remove('active-link', 'bg-purple-700', 'text-white');
                link.classList.add('text-gray-300', 'hover:bg-gray-700');
                if (link.getAttribute('data-page') === activePage) {
                    link.classList.add('active-link', 'bg-purple-700', 'text-white');
                    link.classList.remove('text-gray-300', 'hover:bg-gray-700');
                }
            });
            currentPage = activePage;
        }

        // --- AUTH / PROFILE Functions ---

        async function fetchUserMetadata(uid, authUser) {
            const userDocRef = getUserProfileDocRef(uid);
            const userDocSnap = await getDoc(userDocRef);
            const defaultName = authUser.displayName || authUser.email || `Usuario_${uid.substring(0, 8)}`;

            if (userDocSnap.exists()) {
                userDisplayName = userDocSnap.data().displayName || defaultName;
            } else {
                userDisplayName = defaultName;
                await setDoc(userDocRef, { displayName: defaultName, email: authUser.email || null, uid: uid }, { merge: true });
            }
            if (currentPage === 'profile') renderProfile();
            console.log("Nombre de usuario establecido:", userDisplayName);
        }
        
        // Función para cambiar el nombre de usuario
        window.handleUpdateDisplayName = async () => {
            const newNameInput = document.getElementById('new-display-name');
            const newName = newNameInput.value.trim();
            if (!newName || !userId || !db) {
                showModal("Error", "El nombre no puede estar vacío.");
                return;
            }

            const saveBtn = document.getElementById('save-name-btn');
            saveBtn.disabled = true;
            saveBtn.textContent = 'Guardando...';

            try {
                const userDocRef = getUserProfileDocRef(userId);
                await setDoc(userDocRef, { displayName: newName }, { merge: true });
                userDisplayName = newName;
                showModal("Éxito", "Tu nombre de usuario se ha actualizado correctamente.");
            } catch (error) {
                console.error("Error al actualizar el nombre:", error);
                showModal("Error", "No se pudo actualizar el nombre. Intenta de nuevo.");
            } finally {
                saveBtn.disabled = false;
                saveBtn.textContent = 'Guardar Nuevo Nombre';
                if (currentPage === 'profile') renderProfile(); // Re-renderizar para reflejar el cambio
            }
        }

        // Función para manejar el intento de inicio de sesión con Google
        window.handleGoogleSignIn = async () => {
            if (!auth) {
                showModal("Error", "Firebase Auth no está inicializado.");
                return;
            }
            const provider = new GoogleAuthProvider();
            try {
                // Debido a las restricciones del entorno (iframe), esto fallará con auth/unauthorized-domain.
                await signInWithPopup(auth, provider);
                // Si funciona, onAuthStateChanged manejará el resto.
            } catch (error) {
                // FIX: El error auth/unauthorized-domain es esperado en este entorno (sandbox).
                // Lo capturamos y mostramos un mensaje, sin loguearlo como ERROR en la consola
                // para reducir el ruido visual al usuario.
                if (error.code === 'auth/unauthorized-domain' || error.code === 'auth/popup-closed-by-user') {
                    console.log("Advertencia esperada de Google Auth en sandbox:", error.code); 
                    showModal(
                        "Acceso Deshabilitado", 
                        "La autenticación con Google no funciona en este entorno seguro (sandbox). Por favor, continúa usando tu sesión anónima/temporal activa, la cual permite publicar historias."
                    );
                } else {
                    console.error("Error de inicio de sesión con Google:", error);
                    showModal("Error Desconocido", "Ocurrió un problema inesperado durante la autenticación. Por favor, verifica la consola para detalles.");
                }
            }
        };

        // --- STORY CREATION Functions ---

        // Lógica de tags
        window.toggleTag = (tag) => {
            const index = selectedTags.indexOf(tag);
            const tagElement = document.getElementById(`tag-${tag.replace(/\s/g, '_')}`);
            
            if (index > -1) {
                selectedTags.splice(index, 1);
                if(tagElement) tagElement.classList.remove('tag-selected');
            } else if (selectedTags.length < 10) { // Límite de 10 etiquetas
                selectedTags.push(tag);
                if(tagElement) tagElement.classList.add('tag-selected');
            } else {
                showModal("Límite de Etiquetas", "Solo puedes seleccionar un máximo de 10 etiquetas por historia (incluyendo personalizadas).");
                return;
            }
            updateSelectedTagsDisplay();
        };

        function updateSelectedTagsDisplay() {
            const display = document.getElementById('selected-tags-display');
            if (display) {
                display.textContent = selectedTags.length > 0 
                    ? 'Etiquetas seleccionadas: ' + selectedTags.join(', ')
                    : 'Etiquetas seleccionadas: Ninguna';
            }
        }

        // Lógica de previsualización de imagen
        window.handleFileSelect = (event) => {
            const file = event.target.files[0];
            const fileNameDisplay = document.getElementById('file-name-display');
            const previewImg = document.getElementById('cover-preview-img');

            if (file) {
                fileNameDisplay.textContent = file.name;
                const reader = new FileReader();
                reader.onload = function(e) {
                    coverPreviewUrl = e.target.result;
                    previewImg.src = coverPreviewUrl;
                    previewImg.classList.remove('hidden');
                    document.getElementById('cover-upload-area').classList.add('hidden');
                };
                reader.readAsDataURL(file);
            } else {
                fileNameDisplay.textContent = 'Ningún archivo seleccionado';
                coverPreviewUrl = null;
                previewImg.classList.add('hidden');
                document.getElementById('cover-upload-area').classList.remove('hidden');
            }
        };

        // Función para mostrar información sobre la gestión de capítulos
        window.showChapterInfo = () => {
            showModal(
                "Gestión de Capítulos",
                "Actualmente, cada publicación se considera el 'Capítulo 1'.\n\nEl siguiente paso en el desarrollo sería una vista de 'Edición' en 'Mis Historias' que te permita añadir nuevos capítulos y actualizar el contenido. ¡Pide esta funcionalidad en el chat si la necesitas!"
            );
        };

        // --- Vistas de la Aplicación ---

        // 1. Inicio
        function renderHome() {
            renderContent(`
                <section class="text-center py-16 bg-gray-800 rounded-2xl shadow-2xl">
                    <h2 class="text-5xl font-extrabold text-white mb-4">Bienvenido, <span class="text-purple-400">${userDisplayName}</span></h2>
                    <p class="text-xl text-gray-400 mb-8 max-w-2xl mx-auto">
                        Tu espacio para dar vida a tus ideas. Crea, comparte y explora historias increíbles de nuestra comunidad.
                    </p>
                    <button onclick="changePage('create')" class="bg-purple-600 hover:bg-purple-700 text-white font-bold py-3 px-8 rounded-xl shadow-lg transform hover:scale-105 transition duration-300 ease-in-out flex items-center justify-center mx-auto w-fit">
                        <i data-lucide="quill" class="w-5 h-5 mr-2"></i>
                        Comenzar a Escribir
                    </button>
                    <p class="mt-8 text-sm text-gray-500">
                        Estado de conexión: <span class="${isAuthReady ? 'text-green-400' : 'text-yellow-400'}">${isAuthReady ? 'Conectado' : 'Cargando...'}</span>
                    </p>
                </section>
            `);
        }

        // 2. Crear Historia
        function renderCreateStory() {
            selectedTags = []; // Resetear tags
            // No reseteamos coverPreviewUrl aquí, se hace en el handleFileSelect si el usuario quita el archivo.

            // Generar HTML de etiquetas predefinidas
            const tagsHtml = TAGS.map(tag => `
                <button type="button" id="tag-${tag.replace(/\s/g, '_')}" 
                    onclick="toggleTag('${tag}')" 
                    class="px-3 py-1 text-xs font-medium rounded-full border border-gray-600 bg-gray-900 text-gray-300 hover:bg-purple-800 hover:border-purple-500 transition duration-150">
                    ${tag}
                </button>
            `).join('');


            renderContent(`
                <section class="max-w-4xl mx-auto bg-gray-800 p-8 rounded-2xl shadow-2xl">
                    <h2 class="text-3xl font-bold text-purple-400 mb-6">Nueva Historia y Primer Capítulo</h2>
                    <form id="story-form" class="space-y-6">
                        <div class="flex flex-col md:flex-row gap-6">
                            <!-- Columna Izquierda: Imagen de Portada -->
                            <div class="flex-shrink-0 w-full md:w-1/3 space-y-3">
                                <label class="block text-sm font-medium text-gray-300">Portada de la Historia</label>
                                
                                <!-- Área de Previsualización y Carga -->
                                <!-- Si coverPreviewUrl existe, la mostramos, sino, oculta -->
                                <img id="cover-preview-img" 
                                     src="${coverPreviewUrl || ''}" 
                                     class="w-full aspect-[2/3] object-cover rounded-lg shadow-xl ${coverPreviewUrl ? '' : 'hidden'}" 
                                     alt="Vista previa de la portada">

                                <label for="story-cover-file" id="cover-upload-area" 
                                       class="w-full aspect-[2/3] bg-gray-700 rounded-lg border-2 border-dashed border-gray-600 flex flex-col items-center justify-center cursor-pointer hover:border-purple-500 transition duration-150 p-4 ${coverPreviewUrl ? 'hidden' : ''}">
                                    <i data-lucide="image" class="w-10 h-10 text-gray-400 mb-2"></i>
                                    <p class="text-gray-400 text-sm text-center">Haz clic para elegir una imagen de tu PC</p>
                                    <p class="text-xs text-gray-500 mt-1">¡ADVERTENCIA! Solo se previsualiza. Usa URL para guardar.</p>
                                </label>
                                
                                <!-- Input de Archivo (Oculto) -->
                                <input type="file" id="story-cover-file" accept="image/*" class="hidden" onchange="handleFileSelect(event)">
                                <p id="file-name-display" class="text-purple-400 text-xs mt-1 truncate max-w-full px-2 text-center">Ningún archivo seleccionado</p>
                                
                                <!-- Input de URL de Imagen (Método de guardado) -->
                                <input type="url" id="story-image-url" class="w-full px-4 py-2 bg-gray-700 text-white border border-gray-600 rounded-lg focus:border-purple-500 focus:ring-purple-500 transition duration-150 text-sm" placeholder="O pega aquí una URL de imagen (Opcional)">

                            </div>
                            
                            <!-- Columna Derecha: Detalles y Contenido -->
                            <div class="w-full md:w-2/3 space-y-6">
                                <!-- Título de la Historia -->
                                <div>
                                    <label for="story-title" class="block text-sm font-medium text-gray-300 mb-1">Título de la Historia</label>
                                    <input type="text" id="story-title" required class="w-full px-4 py-3 bg-gray-700 text-white border border-gray-600 rounded-lg focus:border-purple-500 focus:ring-purple-500 transition duration-150" placeholder="Un título memorable">
                                </div>
                                
                                <!-- Tags/Etiquetas Predefinidas -->
                                <div>
                                    <label class="block text-sm font-medium text-gray-300 mb-2">Etiquetas Predefinidas (Máx 10)</label>
                                    <div id="tag-selector" class="flex flex-wrap gap-2 p-3 bg-gray-700 rounded-lg max-h-40 overflow-y-auto">
                                        ${tagsHtml}
                                    </div>
                                </div>

                                <!-- Etiquetas Personalizadas -->
                                <div>
                                    <label for="custom-tags-input" class="block text-sm font-medium text-gray-300 mb-1">Etiquetas Personalizadas (separadas por coma)</label>
                                    <input type="text" id="custom-tags-input" class="w-full px-4 py-3 bg-gray-700 text-white border border-gray-600 rounded-lg focus:border-purple-500 focus:ring-purple-500 transition duration-150" placeholder="Ej: Fantasía Oscura, Dragones, Magia">
                                </div>

                                <div id="selected-tags-display" class="mt-2 text-sm text-purple-400">Etiquetas seleccionadas: Ninguna</div>

                                <!-- Contenido/Capítulo 1 -->
                                <div>
                                    <div class="flex items-center justify-between mb-1">
                                        <label for="story-content" class="block text-sm font-medium text-gray-300">Capítulo 1: Contenido (Descripción)</label>
                                        <button type="button" onclick="showChapterInfo()" class="text-indigo-400 hover:text-indigo-300 transition duration-150" title="Agregar más capítulos">
                                            <i data-lucide="plus-circle" class="w-6 h-6"></i>
                                        </button>
                                    </div>
                                    <textarea id="story-content" rows="8" required class="w-full px-4 py-3 bg-gray-700 text-white border border-gray-600 rounded-lg focus:border-purple-500 focus:ring-purple-500 transition duration-150" placeholder="Empieza a tejer tu mundo..."></textarea>
                                </div>

                                <!-- Botón de Publicar -->
                                <button type="submit" id="submit-btn" class="w-full bg-green-600 hover:bg-green-700 text-white font-bold py-3 rounded-lg shadow-md transition duration-150 flex items-center justify-center">
                                    <i data-lucide="send" class="w-5 h-5 mr-2"></i>
                                    Publicar Historia (Primer Capítulo)
                                </button>
                            </div>
                        </div>
                    </form>
                </section>
            `);
            
            document.getElementById('story-form').addEventListener('submit', handleSaveStory);
            updateSelectedTagsDisplay();
        }

        async function handleSaveStory(event) {
            event.preventDefault();
            if (!userId) {
                showModal("Error de Autenticación", "Necesitas estar autenticado para publicar. Intenta recargar la página.");
                return;
            }

            const title = document.getElementById('story-title').value.trim();
            const content = document.getElementById('story-content').value.trim();
            
            const urlInput = document.getElementById('story-image-url').value.trim();
            let finalCoverImageUrl;

            // --- FIX: Prevent saving large Base64 string to Firestore ---
            if (urlInput) {
                // Prioridad 1: Usar URL pública si el usuario la proporcionó
                finalCoverImageUrl = urlInput;
            } else if (coverPreviewUrl && coverPreviewUrl.startsWith('data:')) {
                // Prioridad 2: Si es una imagen local (Base64), NO GUARDAR EL BASE64. Usar placeholder y advertir.
                finalCoverImageUrl = 'https://placehold.co/400x600/6b46c1/ffffff?text=Imagen+Local+-+URL+Requerida';
                showModal("Advertencia de Imagen", "La imagen local seleccionada no puede ser guardada en la base de datos debido a su tamaño. Se usará una imagen por defecto. Por favor, introduce una URL pública para que la portada sea visible.");
            } else {
                // Prioridad 3: Usar placeholder por defecto
                finalCoverImageUrl = 'https://placehold.co/400x600/6b46c1/ffffff?text=Narrativa+Cover';
            }
            // --- END FIX ---


            // Lógica para etiquetas personalizadas
            const customTagsInput = document.getElementById('custom-tags-input').value.trim();
            const customTagsArray = customTagsInput ? customTagsInput.split(',').map(tag => tag.trim()).filter(tag => tag.length > 0) : [];
            
            // Combinar tags predefinidos y personalizados, limitando a 10 en total.
            const finalTags = [...new Set([...selectedTags, ...customTagsArray])].slice(0, 10);
            
            const submitBtn = document.getElementById('submit-btn');

            if (!title || !content) {
                showModal("Campos Requeridos", "Por favor, completa tanto el título como el contenido de la historia.");
                return;
            }

            submitBtn.disabled = true;
            submitBtn.textContent = 'Publicando...';
            submitBtn.classList.remove('bg-green-600', 'hover:bg-green-700');
            submitBtn.classList.add('bg-gray-500');

            try {
                await addDoc(collection(db, STORIES_COLLECTION_PATH), {
                    title: title,
                    content: content,
                    coverImageUrl: finalCoverImageUrl, 
                    tags: finalTags,
                    chapters: 1, // Simulación de 1 capítulo
                    authorId: userId,
                    authorDisplayId: userDisplayName, // Usamos el nombre de usuario actual
                    timestamp: serverTimestamp()
                });

                showModal("¡Éxito!", "Tu historia ha sido publicada exitosamente.");
                document.getElementById('story-form').reset(); 
                selectedTags = []; 
                coverPreviewUrl = null;
                window.changePage('my-stories'); // Navegar a Mis Historias
                
            } catch (e) {
                console.error("Error al añadir documento: ", e);
                showModal("Error de Base de Datos", "No se pudo publicar la historia. Revisa la consola para más detalles.");
            } finally {
                submitBtn.disabled = false;
                submitBtn.textContent = 'Publicar Historia (Primer Capítulo)';
                submitBtn.classList.remove('bg-gray-500');
                submitBtn.classList.add('bg-green-600', 'hover:bg-green-700');
            }
        }

        // 3. Explorar (Feed Público)
        function renderExploreFeed(stories) {
            let storiesHtml = '';
            if (stories.length === 0) {
                storiesHtml = `<p class="text-center text-gray-500 text-lg py-12">Aún no hay historias publicadas. ¡Sé el primero en crear una!</p>`;
            } else {
                storiesHtml = stories.map(story => `
                    <div class="bg-gray-800 p-6 rounded-xl shadow-lg border-t-4 border-purple-500 flex flex-col md:flex-row gap-6">
                        
                        <!-- Imagen de Portada -->
                        <img src="${story.coverImageUrl || 'https://placehold.co/400x600/6b46c1/ffffff?text=Narrativa+Cover'}" 
                             onerror="this.onerror=null; this.src='https://placehold.co/400x600/6b46c1/ffffff?text=Narrativa+Cover'" 
                             class="w-full md:w-32 aspect-[2/3] object-cover rounded-lg shadow-md flex-shrink-0" alt="Portada de la Historia">
                        
                        <!-- Contenido -->
                        <div>
                            <h3 class="text-2xl font-bold text-white mb-2">${story.title}</h3>
                            <p class="text-sm text-gray-400 mb-2">Por: <span class="font-semibold text-purple-300">@${story.authorDisplayId}</span> - Publicado: ${story.date}</p>
                            
                            <!-- Etiquetas -->
                            <div class="flex flex-wrap gap-1 mb-4">
                                ${(story.tags || []).map(tag => 
                                    `<span class="px-2 py-0.5 text-xs font-semibold rounded-full bg-purple-900 text-purple-300 border border-purple-700">${tag}</span>`
                                ).join('')}
                            </div>
                            
                            <div class="text-gray-300 text-base max-h-24 overflow-hidden relative mb-4 story-preview">
                                <p>${story.content.substring(0, 150)}...</p>
                                <div class="absolute bottom-0 left-0 w-full h-8 bg-gradient-to-t from-gray-800 to-transparent"></div>
                            </div>
                            <button onclick="showStoryDetails('${story.id}')" class="text-purple-400 hover:text-purple-300 font-semibold flex items-center">
                                Leer más
                                <i data-lucide="chevron-right" class="w-4 h-4 ml-1"></i>
                            </button>
                        </div>
                    </div>
                `).join('<hr class="border-gray-700 my-6">');
            }

            renderContent(`
                <section>
                    <h2 class="text-4xl font-extrabold text-white mb-8">Explorar Historias de la Comunidad</h2>
                    <div id="story-feed" class="space-y-6 story-feed">
                        ${storiesHtml}
                    </div>
                </section>
            `);
        }

        // Función auxiliar para mostrar detalles de la historia (simulando lectura)
        window.showStoryDetails = (storyId) => {
            const story = allStories.find(s => s.id === storyId);
            if (story) {
                const tagsList = (story.tags && story.tags.length > 0) ? `\n\nEtiquetas: ${story.tags.join(', ')}` : '';
                showModal(
                    `Leyendo: ${story.title}`, 
                    `Autor: ${story.authorDisplayId}\nCapítulos: ${story.chapters || 1}\n${tagsList}\n\n${story.content.substring(0, 1000)}... (Texto completo simulado)`
                );
            } else {
                showModal("Error", "Historia no encontrada.");
            }
        };

        // 4. Mis Historias (Actualizado con vista detallada)
        function renderMyStories(stories) {
            const myStories = stories.filter(story => story.authorId === userId);
            let storiesHtml = '';
            if (myStories.length === 0) {
                storiesHtml = `<p class="text-center text-gray-500 text-lg py-12">Aún no has publicado ninguna historia. ¡Anímate a escribir!</p>`;
            } else {
                storiesHtml = myStories.map(story => `
                    <div class="bg-gray-800 p-6 rounded-xl shadow-lg border-l-4 border-yellow-500 flex flex-col sm:flex-row gap-6">
                        
                        <!-- Imagen de Portada -->
                        <img src="${story.coverImageUrl || 'https://placehold.co/400x600/6b46c1/ffffff?text=Narrativa+Cover'}" 
                             onerror="this.onerror=null; this.src='https://placehold.co/400x600/6b46c1/ffffff?text=Narrativa+Cover'" 
                             class="w-full sm:w-24 aspect-[2/3] object-cover rounded-lg shadow-md flex-shrink-0" alt="Portada de la Historia">
                        
                        <!-- Contenido y Detalles -->
                        <div class="flex-grow">
                            <h3 class="text-2xl font-bold text-white mb-1">${story.title}</h3>
                            <p class="text-sm text-yellow-400 mb-2">${story.chapters || 1} Capítulo publicado</p>
                            
                            <!-- Resumen/Descripción -->
                            <p class="text-gray-300 text-base max-h-16 overflow-hidden relative mb-3">
                                ${story.content.substring(0, 100)}...
                            </p>

                            <!-- Etiquetas/Prompts -->
                            <div class="flex flex-wrap gap-1 mb-4">
                                ${(story.tags || []).map(tag => 
                                    `<span class="px-2 py-0.5 text-xs font-semibold rounded-full bg-purple-900 text-purple-300 border border-purple-700">${tag}</span>`
                                ).join('')}
                            </div>
                            
                            <button onclick="showStoryDetails('${story.id}')" class="bg-purple-600 hover:bg-purple-700 text-white font-semibold py-1.5 px-4 rounded-lg transition duration-150 text-sm">
                                Editar / Ver Detalles
                            </button>
                        </div>
                    </div>
                `).join('<hr class="border-gray-700 my-4">');
            }

            renderContent(`
                <section>
                    <h2 class="text-4xl font-extrabold text-white mb-8">Mis Historias Publicadas</h2>
                    <p class="text-gray-400 mb-6">Gestiona tus creaciones. Total de historias: ${myStories.length}</p>
                    <div id="my-stories-list" class="space-y-4">
                        ${storiesHtml}
                    </div>
                </section>
            `);
        }
        
        // 5. Perfil (Actualizado)
        function renderProfile() {
             renderContent(`
                <section class="max-w-xl mx-auto bg-gray-800 p-8 rounded-2xl shadow-2xl text-center">
                    <i data-lucide="user-circle" class="w-16 h-16 text-purple-400 mx-auto mb-4"></i>
                    <h2 class="text-3xl font-bold text-white mb-2">Hola, <span class="text-purple-400">${userDisplayName}</span></h2>
                    <p class="text-gray-400 mb-8">
                        Aquí puedes gestionar tu identidad de autor y estado de sesión.
                    </p>
                    
                    <!-- Formulario de Cambio de Nombre -->
                    <div class="bg-gray-700 p-4 rounded-lg space-y-3 mb-8">
                        <label for="new-display-name" class="block text-sm font-medium text-gray-300 text-left">Cambiar Nombre de Autor</label>
                        <input type="text" id="new-display-name" value="${userDisplayName}" class="w-full px-4 py-2 bg-gray-900 text-white border border-gray-600 rounded-lg focus:border-purple-500 transition duration-150" placeholder="Nuevo nombre de usuario" />
                        <button onclick="handleUpdateDisplayName()" id="save-name-btn" class="w-full bg-yellow-600 hover:bg-yellow-700 text-white font-bold py-2 rounded-lg transition duration-150">
                            Guardar Nuevo Nombre
                        </button>
                    </div>

                    <!-- Botón de Google Sign-In -->
                    <button onclick="handleGoogleSignIn()" class="mt-4 w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-lg shadow-md transition duration-150 flex items-center justify-center">
                        <i data-lucide="log-in" class="w-5 h-5 mr-2"></i>
                        Vincular / Iniciar Sesión con Google
                    </button>
                    
                    <p class="text-xs text-gray-500 mt-8 break-all">
                        ID de Sesión Técnica: ${userId ? userId : 'Cargando...'}
                    </p>
                </section>
            `);
        }

        // --- Lógica de Firebase y Base de Datos (onSnapshot) ---

        let allStories = [];

        function startStoriesListener() {
            const storiesRef = collection(db, STORIES_COLLECTION_PATH);

            onSnapshot(storiesRef, (snapshot) => {
                allStories = snapshot.docs.map(doc => {
                    const data = doc.data();
                    const date = data.timestamp ? new Date(data.timestamp.seconds * 1000).toLocaleDateString('es-ES') : 'N/A';
                    return {
                        id: doc.id,
                        title: data.title,
                        content: data.content,
                        coverImageUrl: data.coverImageUrl,
                        tags: data.tags,
                        chapters: data.chapters || 1,
                        authorId: data.authorId,
                        authorDisplayId: data.authorDisplayId || 'Anónimo',
                        date: date
                    };
                }).sort((a, b) => {
                    const dateA = a.timestamp ? a.timestamp.seconds : 0;
                    const dateB = b.timestamp ? b.timestamp.seconds : 0;
                    return dateB - dateA;
                }); 

                // Vuelve a renderizar la página activa para mostrar los datos actualizados
                if (currentPage === 'explore') renderExploreFeed(allStories);
                if (currentPage === 'my-stories') renderMyStories(allStories);
                
                console.log(`[Firestore] Historias actualizadas: ${allStories.length}`);

            }, (error) => {
                console.error("Error al escuchar historias:", error);
                showModal("Error de Sincronización", "Hubo un problema al cargar las historias en tiempo real.");
            });
        }

        // --- Inicialización y Router ---

        async function initializeAppAndAuth() {
            try {
                if (Object.keys(firebaseConfig).length === 0) {
                     throw new Error("La configuración de Firebase está vacía.");
                }
                setLogLevel('Debug');
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                
                await new Promise((resolve) => {
                    onAuthStateChanged(auth, async (user) => {
                        if (user) {
                            userId = user.uid;
                            await fetchUserMetadata(user.uid, user); // Obtener el nombre de usuario de Firestore/Auth
                        } else {
                            if (initialAuthToken) {
                                await signInWithCustomToken(auth, initialAuthToken);
                                userId = auth.currentUser.uid;
                                await fetchUserMetadata(auth.currentUser.uid, auth.currentUser);
                            } else {
                                const anonUser = await signInAnonymously(auth);
                                userId = anonUser.user.uid;
                                await fetchUserMetadata(anonUser.user.uid, anonUser.user);
                            }
                        }
                        isAuthReady = true;
                        resolve();
                    });
                });

                startStoriesListener();

            } catch (error) {
                console.error("Error crítico de inicialización:", error);
                showModal("Error Crítico", "No se pudo conectar con la base de datos de la plataforma.");
                isAuthReady = true;
                userId = crypto.randomUUID();
                userDisplayName = 'Usuario Desconectado';
            }

            changePage('home');
        }

        // Función global para manejar el cambio de vista
        window.changePage = (page) => {
            updateNavBar(page);
            switch(page) {
                case 'explore':
                    renderExploreFeed(allStories);
                    break;
                case 'create':
                    renderCreateStory();
                    break;
                case 'my-stories':
                    renderMyStories(allStories);
                    break;
                case 'profile':
                    renderProfile();
                    break;
                case 'home':
                default:
                    renderHome();
                    break;
            }
        };

        // Asignar eventos de navegación
        document.addEventListener('DOMContentLoaded', () => {
            navLinks.forEach(link => {
                link.addEventListener('click', () => {
                    window.changePage(link.getAttribute('data-page'));
                });
            });

            initializeAppAndAuth();
        });

    </script>
</body>
</html>
