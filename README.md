# silver-goggles
App for my first business 
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>HSE.Pk</title>

    <!-- PWA Configuration -->
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black" />
    <meta name="apple-mobile-web-app-title" content="HSE.Pk" />
    <meta name="theme-color" content="#172338" />

    <!-- External Libraries -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&family=Caveat:wght@400;700&display=swap" rel="stylesheet" />
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- Firebase SDKs (Required for App Logic) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, createUserWithEmailAndPassword, onAuthStateChanged, signOut, updateProfile } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, query, serverTimestamp, addDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { createElement as e, useState, useEffect, createContext, useContext, lazy, Suspense, useRef } from "https://esm.sh/react@18.2.0";
        import { createRoot } from "https://esm.sh/react-dom@18.2.0/client";

        // ====================================================================================
        // THIS SECTION IS CRITICAL: Reading Environment Variables
        // Vercel injects ENV variables via process.env in build time. 
        // We use common React conventions (REACT_APP_) for ease of Vercel acceptance.
        // ====================================================================================

        // Default/Mock Firebase Config (Used only if ENV variables fail)
        const MOCK_FIREBASE_CONFIG = {
            apiKey: "MOCK_API_KEY",
            authDomain: "MOCK_AUTH_DOMAIN",
            projectId: "MOCK_PROJECT_ID",
            storageBucket: "MOCK_STORAGE_BUCKET",
            messagingSenderId: "MOCK_SENDER_ID",
            appId: "MOCK_APP_ID"
        };
        const MOCK_APP_ID = 'default-app-id';

        let finalFirebaseConfig;
        let finalAppId;

        try {
            // Attempt to load Vercel Environment Variables
            // We assume the user has set REACT_APP_APP_ID and REACT_APP_FIREBASE_CONFIG in Vercel
            
            // Note: In Vercel, process.env.REACT_APP_... is often used for static file injection.
            // We use a safe check here. If the global variable exists (in deployed Vercel) and is valid, use it.
            
            // Check for App ID
            if (typeof process !== 'undefined' && process.env.REACT_APP_APP_ID && process.env.REACT_APP_APP_ID !== 'undefined') {
                finalAppId = process.env.REACT_APP_APP_ID;
            } else {
                finalAppId = MOCK_APP_ID;
            }

            // Check for Firebase Config JSON
            if (typeof process !== 'undefined' && process.env.REACT_APP_FIREBASE_CONFIG && process.env.REACT_APP_FIREBASE_CONFIG !== 'undefined') {
                // IMPORTANT: Must parse the config string into a JSON object
                finalFirebaseConfig = JSON.parse(process.env.REACT_APP_FIREBASE_CONFIG);
            } else {
                finalFirebaseConfig = MOCK_FIREBASE_CONFIG;
            }
            
            // If the Vercel variables are undefined, the app will use MOCK values and crash Firebase,
            // which is intentional to force the user to set the variables.

        } catch (error) {
            console.error("Critical: Failed to load/parse Firebase ENV vars. Using mock data.", error);
            finalFirebaseConfig = MOCK_FIREBASE_CONFIG;
            finalAppId = MOCK_APP_ID;
        }

        // --- Firebase Initialization ---
        const app = initializeApp(finalFirebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);

        // --- Contexts ---
        const AuthContext = createContext();

        // Admin Email Address for Role Assignment
        const ADMIN_EMAIL = "fiazsk.eneabler@gmail.com";

        // --- Utility Functions ---

        const getOrCreateUserDoc = async (user) => {
            const userRef = doc(db, `artifacts/${finalAppId}/users/${user.uid}/profile/details`);
            let role = 'customer';

            if (user.email === ADMIN_EMAIL) {
                role = 'admin';
            }

            try {
                await setDoc(userRef, {
                    email: user.email,
                    uid: user.uid,
                    role: role,
                    lastLogin: serverTimestamp(),
                    displayName: user.email === ADMIN_EMAIL ? "Fayaz Urrehman" : user.email.split('@')[0]
                }, { merge: true });
                return { uid: user.uid, email: user.email, role: role };
            } catch (error) {
                console.error("Error setting user document:", error);
                return { uid: user.uid, email: user.email, role: 'visitor' };
            }
        };

        // --- Auth Provider Component ---

        const AuthProvider = ({ children }) => {
            const [user, setUser] = useState(null);
            const [profile, setProfile] = useState(null);
            const [loading, setLoading] = useState(true);

            useEffect(() => {
                const unsubscribe = onAuthStateChanged(auth, async (currentUser) => {
                    if (currentUser) {
                        const userRef = doc(db, `artifacts/${finalAppId}/users/${currentUser.uid}/profile/details`);
                        const unsubscribeProfile = onSnapshot(userRef, (docSnap) => {
                            if (docSnap.exists()) {
                                setProfile(docSnap.data());
                            }
                            setUser(currentUser);
                            setLoading(false);
                        }, (error) => {
                            console.error("Error fetching user profile:", error);
                            setProfile({ role: 'customer' });
                            setUser(currentUser);
                            setLoading(false);
                        });
                        return () => unsubscribeProfile();
                    } else {
                        setUser(null);
                        setProfile({ role: 'visitor' });
                        setLoading(false);
                    }
                });
                return () => unsubscribe();
            }, []);

            const login = async (email, password) => {
                const userCredential = await signInWithEmailAndPassword(auth, email, password);
                await getOrCreateUserDoc(userCredential.user);
                return userCredential;
            };

            const signup = async (email, password) => {
                const userCredential = await createUserWithEmailAndPassword(auth, email, password);
                await updateProfile(userCredential.user, {
                    displayName: email.split('@')[0]
                });
                await getOrCreateUserDoc(userCredential.user);
                return userCredential;
            };

            const logout = () => signOut(auth);

            const value = { user, profile, loading, login, signup, logout };

            return e(AuthContext.Provider, { value }, children);
        };

        const useAuth = () => useContext(AuthContext);

        // --- PWA Service Worker Registration (Emulation) ---
        useEffect(() => {
            if ('serviceWorker' in navigator) {
                navigator.serviceWorker.register('data:application/javascript;base64,' + btoa(
                    "self.addEventListener('fetch', function(event) {}); self.addEventListener('install', function(event) {self.skipWaiting();}); self.addEventListener('activate', function(event) { event.waitUntil(clients.claim()); });"
                ), { scope: '/' })
                .then(registration => console.log('SW registration successful:', registration.scope))
                .catch(error => console.error('SW registration failed:', error));
            }
        }, []);

        // --- LAZY LOADED COMPONENTS & MOCKS ---
        const createLazyMock = (component) => lazy(() => new Promise(resolve => resolve({ default: component })));

        const LoadingSpinner = () => (
            e('div', {className: "flex justify-center items-center h-full p-10"},
                e('div', {className: "animate-spin rounded-full h-8 w-8 border-b-2 border-indigo-500"}),
                e('p', {className: "ml-3 text-gray-500"}, "Loading content segment...")
            )
        );
        
        // Component Definitions (Full code is condensed for single-file deployment)
        const AdminDashboardComponent = () => { 
            return e('div', {className:"p-6 md:p-8"}, e('h2', {className:"text-2xl font-bold text-indigo-500 mb-4"}, "Admin Dashboard (SK. HSE.)"), e('p', {className:"text-gray-400 mb-6"}, "Welcome, Fayaz Urrehman (Admin). Here you can manage orders, view analytics, and respond to feedback."), e('div', {className:"grid grid-cols-1 md:grid-cols-3 gap-6"}, e('div', {className:"bg-gray-700 p-4 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300"}, e('p', {className:"text-indigo-400 font-semibold text-lg"}, "New Orders"), e('p', {className:"text-3xl text-white mt-1"}, "14"), e('p', {className:"text-xs text-gray-400 mt-2"}, "Since Last Login")), e('div', {className:"bg-gray-700 p-4 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300"}, e('p', {className:"text-indigo-400 font-semibold text-lg"}, "Total Customers"), e('p', {className:"text-3xl text-white mt-1"}, "210"), e('p', {className:"text-xs text-gray-400 mt-2"}, "Customers Signed Up")), e('div', {className:"bg-gray-700 p-4 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300"}, e('p', {className:"text-indigo-400 font-semibold text-lg"}, "Avg. Feedback"), e('p', {className:"text-3xl text-white mt-1"}, "4.8"), e('p', {className:"text-xs text-gray-400 mt-2"}, "Overall Rating (Last 30 Days)"))), e('div', {className:"mt-8"}, e('p', {className:"text-indigo-400 text-xl font-semibold mb-3 border-b border-indigo-700 pb-2"}, "Admin Tools (Simulated)"), e('button', {className:"bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 px-4 rounded-lg mr-3 shadow-md transition duration-300"}, "Review Orders"), e('button', {className:"bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300"}, "Manage Products")))}
        const HomeViewComponent = () => { 
            const { profile } = useAuth();
            const roleText = profile?.role === 'admin' ? 'Admin' : profile?.role === 'customer' ? 'Customer' : 'Visitor';
            return e('div', {className:"p-6 md:p-8"}, e('h2', {className:"text-2xl font-bold text-indigo-500 mb-2"}, "Welcome to SK. HSE."), e('p', {className:"text-gray-400 mb-6"}, "Your status: ", e('span', {className:`font-semibold ${roleText === 'Admin' ? 'text-indigo-300' : 'text-gray-300'}`}, roleText), ". We are your trusted partner for premium global exports."), e('div', {className:"grid grid-cols-1 md:grid-cols-3 gap-6"}, e('div', {className:"p-4 bg-gray-800 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300 transform hover:scale-[1.01]"}, e('div', {className:"text-indigo-400 text-3xl mb-3"}, "📦"), e('h3', {className:"font-bold text-lg text-white mb-1"}, "Unmatched Quality"), e('p', {className:"text-sm text-gray-400"}, "Rigorous sourcing from the Karakoram region ensures excellence in every export.")), e('div', {className:"p-4 bg-gray-800 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300 transform hover:scale-[1.01]"}, e('div', {className:"text-indigo-400 text-3xl mb-3"}, "✈️"), e('h3', {className:"font-bold text-lg text-white mb-1"}, "Global Logistics"), e('p', {className:"text-sm text-gray-400"}, "Reliable, timely, and secure shipping worldwide, backed by transparent tracking.")), e('div', {className:"p-4 bg-gray-800 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300 transform hover:scale-[1.01]"}, e('div', {className:"text-indigo-400 text-3xl mb-3"}, "💬"), e('h3', {className:"font-bold text-lg text-white mb-1"}, "Direct Communication"), e('p', {className:"text-sm text-gray-400"}, "Real-time support and chat access for immediate assistance and query resolution.")) ), profile?.role === 'admin' && e('div', {className:"mt-8 p-4 bg-indigo-900/50 rounded-xl shadow-lg"}, e('h3', {className:"text-xl font-semibold text-indigo-300"}, "Admin Quick Access"), e('p', {className:"text-sm text-gray-400"}, "Use the Admin Dashboard to manage your operations.")) )}
        const AboutViewComponent = () => { 
            const ContactInfo = ({ icon, label, value }) => (e('div', {className:"flex items-center mb-4"}, e('div', {className:"text-indigo-400 text-2xl mr-3"}, icon), e('div', null, e('p', {className:"text-gray-400 text-sm"}, label), e('p', {className:"text-white font-medium"}, value))))
            return e('div', {className:"p-6 md:p-8 grid grid-cols-1 lg:grid-cols-3 gap-8"}, e('div', {className:"lg:col-span-2"}, e('h3', {className:"text-xl font-bold text-white mb-4 glowing-border-bottom pb-1"}, "Our Vision"), e('p', {className:"text-gray-300 mb-6 text-lg italic"}, "\"Experience the pure joy of product benefits and create memories that last a lifetime.\""), e('p', {className:"text-gray-400 mb-8"}, "We care about your well being. SK. HSE. was founded to bridge the gap between high-quality, sustainably sourced hill-side products and the global market. We specialize in ethically sourced goods, ensuring that every export meets rigorous quality standards from cultivation to final shipment."), e('h3', {className:"text-xl font-bold text-white mb-4 glowing-border-bottom pb-1"}, "Founder's Profile"), e('div', {className:"flex flex-col md:flex-row items-center md:items-start bg-gray-800 p-6 rounded-xl shadow-xl"}, e('img', {src:"https://image.pollinations.ai/prompt/man-sitting-in-snowy-mountains-wearing-shalwar-kameez-and-brown-coat?width=150&height=225&seed=new", alt:"Fayaz Urrehman, Founder of SK. HSE.", className:"w-24 h-24 md:w-32 md:h-32 rounded-full object-cover mb-4 md:mb-0 md:mr-6 border-4 border-indigo-500"}), e('div', null, e('p', {className:"text-2xl text-indigo-400 font-caveat tracking-wider mt-0 pt-0 text-shadow-indigo"}, "Fayaz Urrehman"), e('p', {className:"text-sm font-semibold text-indigo-500 mb-2"}, "Founder & CEO, SK. HSE."), e('p', {className:"text-gray-400 text-sm"}, "Driven by a passion for ethical trade and the rich biodiversity of the Karakoram regions, Fayaz ensures that SK. HSE. delivers only premium, high-altitude products to the world market."))),), e('div', {className:"lg:col-span-1 bg-gray-800 p-6 rounded-xl shadow-xl"}, e('h3', {className:"text-xl font-bold text-white mb-4 glowing-border-bottom pb-1"}, "Get In Touch"), e('p', {className:"text-gray-400 mb-6"}, "We are available 24/7 for export inquiries, partnership proposals, and support."), ContactInfo({icon:"📧", label:"Email Address", value:"fiazsk.eneabler@gmail.com"}), ContactInfo({icon:"📞", label:"Phone / WhatsApp", value:"+92 345 5093655"}), ContactInfo({icon:"📍", label:"Headquarters", value:"Rawalpindi, Pakistan (Global Export Hub)"})));
        }
        const ProductsViewComponent = () => { 
            const products = [{ name: "Qaraqram Salajeet (Shilajit)", details: "Premium, purified mineral resin sourced from extreme high altitudes (4000m+). High in fulvic acid and trace minerals. Exported in bulk glass containers." }, { name: "Organic Hill Coffee Beans", details: "Rare, high-altitude Arabica beans with a rich, low-acid profile. Hand-picked and naturally processed. Certified organic export grade." }, { name: "Walnuts (Kashmiri Variety)", details: "Large, light-colored kernels known for their thin shells and high oil content. Ideal for bulk processing and direct consumer packaging." }, { name: "Wildflower Honey (Unpasteurized)", details: "Harvested from remote floral valleys. Raw, unpasteurized, and fully traceable. Exported in food-grade barrels or customized jars." }, { name: "Dried Wild Morels (Mushrooms)", details: "Highly prized Morchella esculenta variety. Naturally air-dried to retain intense flavor and aroma. Available in Grade A/B export quality." },];
            const [formState, setFormState] = useState({ name: '', email: '', details: '' });
            const [message, setMessage] = useState(null);
            const handleChange = (e) => setFormState({ ...formState, [e.target.name]: e.target.value });
            const handleSubmit = async (e) => { e.preventDefault(); setMessage({ type: 'loading', text: 'Sending quote request...' }); try { const inquiryData = { ...formState, timestamp: serverTimestamp(), status: 'New Inquiry' }; const inquiryPath = `artifacts/${finalAppId}/public/data/price_inquiries`; await addDoc(collection(db, inquiryPath), inquiryData); setMessage({ type: 'success', text: 'Quote request submitted! We will contact you soon.' }); setFormState({ name: '', email: '', details: '' }); } catch (error) { console.error("Error submitting quote request:", error); setMessage({ type: 'error', text: 'Failed to submit request. Please try again.' }); } };
            const ProductCard = ({ name, details }) => e('div', {className:"p-5 bg-gray-800 rounded-xl shadow-lg hover:shadow-indigo-500/30 transition duration-300"}, e('h4', {className:"text-xl font-bold text-white mb-2 flex items-center"}, e('span', {className:"w-8 h-8 rounded-full bg-indigo-600 flex items-center justify-center text-white font-bold text-lg mr-3 shadow-md"}, name.charAt(0)), name), e('p', {className:"text-gray-400 text-sm"}, details));
            return e('div', {className:"p-6 md:p-8 grid grid-cols-1 lg:grid-cols-3 gap-8"}, e('div', {className:"lg:col-span-2 space-y-6"}, e('h3', {className:"text-xl font-bold text-white mb-4 glowing-border-bottom pb-1"}, "Premium Export Products"), products.map((p, index) => e(ProductCard, {key:index, name:p.name, details:p.details}))), e('div', {className:"lg:col-span-1 bg-gray-800 p-6 rounded-xl shadow-xl sticky top-4"}, e('h3', {className:"text-xl font-bold text-white mb-4 glowing-border-bottom pb-1"}, "Request Custom Pricing"), e('p', {className:"text-gray-400 text-sm mb-4"}, "Tell us your required volume and destination for a precise, best-rate quote."), e('form', {onSubmit:handleSubmit}, e('input', {type:"text", name:"name", value:formState.name, onChange:handleChange, placeholder:"Your Name/Company", required:true, className:"w-full p-2 mb-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500 focus:ring-indigo-500 shadow-inner"}), e('input', {type:"email", name:"email", value:formState.email, onChange:handleChange, placeholder:"Email Address", required:true, className:"w-full p-2 mb-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500 focus:ring-indigo-500 shadow-inner"}), e('textarea', {name:"details", value:formState.details, onChange:handleChange, placeholder:"Required products, volume, and destination port.", rows:"4", required:true, className:"w-full p-2 mb-4 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-indigo-500 focus:ring-indigo-500 shadow-inner"}), e('button', {type:"submit", className:"w-full py-2 bg-indigo-600 text-white font-semibold rounded-lg shadow-lg shadow-indigo-500/40 hover:bg-indigo-700 transition duration-300"}, "Get Quote")), message && e('div', {className:`mt-4 p-3 rounded-lg font-medium text-center ${message.type === 'success' ? 'bg-indigo-800 text-indigo-200' : message.type === 'error' ? 'bg-red-800 text-red-200' : 'bg-gray-700 text-gray-400'}`}, message.text)));
        }
        const OrderViewComponent = () => { 
            const { profile } = useAuth();
            const [formState, setFormState] = useState({ buyerName: '', contactEmail: '', productDetails: '', shippingAddress: '' });
            const [message, setMessage] = useState(null);
            if (profile?.role === 'visitor') { return e('div', {className:"p-6 md:p-8 text-center text-red-400"}, "Please login or sign up as a Customer to place an order."); }
            const handleChange = (e) => setFormState({ ...formState, [e.target.name]: e.target.value });
            const handleSubmit = async (e) => { e.preventDefault(); setMessage({ type: 'loading', text: 'Submitting order...' }); try { const orderData = { ...formState, userId: profile?.uid, timestamp: serverTimestamp(), status: 'Pending Review' }; const orderPath = `artifacts/${finalAppId}/public/data/orders`; await addDoc(collection(db, orderPath), orderData); setMessage({ type: 'success', text: 'Order submitted successfully! We will contact you shortly.' }); setFormState({ buyerName: '', contactEmail: '', productDetails: '', shippingAddress: '' }); } catch (error) { console.error("Error submitting order:", error); setMessage({ type: 'error', text: 'Failed to submit order. Please try again.' }); } };
            return e('div', {className:"p-6 md:p-8 bg-gray-800 shadow-xl rounded-xl"}, e('h2', {className:"text-2xl font-bold text-indigo-500 mb-6"}, "Place a New Export Order"), e('form', {onSubmit:handleSubmit}, e('div', {className:"grid grid-cols-1 md:grid-cols-2 gap-4"}, e('div', null, e('label', {htmlFor:"buyerName", className:"block text-sm font-medium text-gray-300 mb-1"}, "Your Name/Company"), e('input', {type:"text", id:"buyerName", name:"buyerName", value:formState.buyerName, onChange:handleChange, required:true, className:"w-full p-2 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), e('div', null, e('label', {htmlFor:"contactEmail", className:"block text-sm font-medium text-gray-300 mb-1"}, "Email Address"), e('input', {type:"email", id:"contactEmail", name:"contactEmail", value:formState.contactEmail, onChange:handleChange, required:true, className:"w-full p-2 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), e('div', {className:"mt-4"}, e('label', {htmlFor:"productDetails", className:"block text-sm font-medium text-gray-300 mb-1"}, "Product Details & Quantity (e.g., 5 tons of Salajeet)"), e('textarea', {id:"productDetails", name:"productDetails", value:formState.productDetails, onChange:handleChange, rows:"3", required:true, className:"w-full p-2 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), e('div', {className:"mt-4"}, e('label', {htmlFor:"shippingAddress", className:"block text-sm font-medium text-gray-300 mb-1"}, "Shipping Destination / Port"), e('textarea', {id:"shippingAddress", name:"shippingAddress", value:formState.shippingAddress, onChange:handleChange, rows:"3", required:true, className:"w-full p-2 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), e('button', {type:"submit", className:"mt-6 w-full py-3 px-4 bg-indigo-600 text-white font-semibold rounded-xl shadow-lg shadow-indigo-500/40 hover:bg-indigo-700 transition duration-300 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2"}, "Submit Order Request")), message && e('div', {className:`mt-4 p-3 rounded-lg font-medium text-center ${message.type === 'success' ? 'bg-indigo-800 text-indigo-200' : message.type === 'error' ? 'bg-red-800 text-red-200' : 'bg-gray-700 text-gray-400'}`}, message.text));
        }
        const FeedbackViewComponent = () => { 
            const { profile } = useAuth();
            const [rating, setRating] = useState(0);
            const [comments, setComments] = useState('');
            const [message, setMessage] = useState(null);
            const [feedbackList, setFeedbackList] = useState([]);
            useEffect(() => { const feedbackPath = `artifacts/${finalAppId}/public/data/feedback`; const feedbackQuery = query(collection(db, feedbackPath)); const unsubscribe = onSnapshot(feedbackQuery, (snapshot) => { const newFeedbackList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })).sort((a, b) => (b.timestamp?.toMillis() || 0) - (a.timestamp?.toMillis() || 0)).slice(0, 5); setFeedbackList(newFeedbackList); }, (error) => { console.error("Error fetching feedback:", error); }); return () => unsubscribe(); }, []);
            if (profile?.role === 'visitor') { return e('div', {className:"p-6 md:p-8 text-center text-red-400"}, "Please login or sign up as a Customer to leave feedback."); }
            const StarIcon = ({ fill, onClick, size = 'w-6 h-6' }) => e('svg', {onClick:onClick, className:`${size} cursor-pointer transition duration-150 ${fill ? 'text-yellow-400' : 'text-gray-600 hover:text-yellow-300'}`, fill:"currentColor", viewBox:"0 0 20 20"}, e('path', {d:"M9.049 2.927c.3-.921 1.638-.921 1.94 0l2.454 7.558 7.947.608c.974.074 1.371 1.344.608 1.996l-6.082 5.093 1.838 8.016c.218.966-.855 1.76-1.745 1.31L10 16.793l-6.908 3.522c-.89.45-1.963-.344-1.745-1.31l1.838-8.016L.058 13.089c-.763-.652-.366-1.922.608-1.996l7.947-.608 2.454-7.558z"}));
            const handleSubmit = async (e) => { e.preventDefault(); if (rating === 0) return setMessage({ type: 'error', text: 'Please select a star rating.' }); setMessage({ type: 'loading', text: 'Submitting feedback...' }); try { const feedbackData = { ratingValue: rating, feedbackComments: comments, userId: profile?.uid, displayName: profile?.displayName || 'Customer', timestamp: serverTimestamp(), }; const feedbackPath = `artifacts/${finalAppId}/public/data/feedback`; await addDoc(collection(db, feedbackPath), feedbackData); setMessage({ type: 'success', text: 'Thank you for your valuable feedback!' }); setRating(0); setComments(''); } catch (error) { console.error("Error submitting feedback:", error); setMessage({ type: 'error', text: 'Failed to submit feedback. Please try again.' }); } };
            const renderRecentFeedback = () => { if (feedbackList.length === 0) { return e('p', {className:"text-gray-500 italic text-center"}, "No recent feedback available."); } return feedbackList.map(item => { const timeString = item.timestamp ? new Date(item.timestamp.toMillis()).toLocaleDateString() : 'N/A'; const comment = item.feedbackComments || "No comment provided."; return e('div', {key:item.id, className:"p-4 bg-gray-700 rounded-lg shadow-inner"}, e('div', {className:"flex items-center justify-between mb-1"}, e('div', {className:"flex"}, Array.from({length: 5}, (_, i) => e(StarIcon, {key:i, fill:i < item.ratingValue, size:"w-4 h-4"}))), e('span', {className:"text-xs text-gray-400"}, timeString)), e('p', {className:"text-sm text-gray-300 italic"}, `"${comment}"`), e('p', {className:"text-xs text-gray-500 mt-1"}, "By: ", item.displayName || item.userId?.substring(0, 8) + '...')); }); };
            return e('div', {className:"p-6 md:p-8 grid grid-cols-1 lg:grid-cols-3 gap-8"}, e('div', {className:"lg:col-span-2 bg-gray-800 shadow-xl rounded-xl p-6"}, e('h2', {className:"text-2xl font-bold text-indigo-500 mb-6"}, "Provide Service Feedback"), e('form', {onSubmit:handleSubmit}, e('div', {className:"mb-4"}, e('label', {className:"block text-sm font-medium text-gray-300 mb-2"}, "Service Rating (1-5 Stars)"), e('div', {className:"flex space-x-1"}, Array.from({length: 5}, (_, i) => e(StarIcon, {key:i, fill:i <= rating, onClick:() => setRating(i)})))), e('div', {className:"mt-4"}, e('label', {htmlFor:"feedbackComments", className:"block text-sm font-medium text-gray-300 mb-1"}, "Comments on our service (optional)"), e('textarea', {id:"feedbackComments", value:comments, onChange:(e) => setComments(e.target.value), rows:"4", className:"w-full p-2 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), e('button', {type:"submit", className:"mt-6 w-full py-3 px-4 bg-indigo-600 text-white font-semibold rounded-xl shadow-lg shadow-indigo-500/40 hover:bg-indigo-700 transition duration-300"}, "Submit Feedback")), message && e('div', {className:`mt-4 p-3 rounded-lg font-medium text-center ${message.type === 'success' ? 'bg-indigo-800 text-indigo-200' : message.type === 'error' ? 'bg-red-800 text-red-200' : 'bg-gray-700 text-gray-400'}`}, message.text)), e('div', {className:"lg:col-span-1 bg-gray-800 p-6 rounded-xl shadow-xl"}, e('h3', {className:"text-xl font-bold text-white mb-4 glowing-border-bottom pb-1"}, "Recent Buyer Feedback"), e('div', {className:"space-y-4"}, renderRecentFeedback())));
        }
        const ChatViewComponent = () => { 
            const { profile } = useAuth();
            const [messages, setMessages] = useState([]);
            const [chatInput, setChatInput] = useState('');
            const messagesEndRef = useRef(null);
            useEffect(() => { const chatPath = `artifacts/${finalAppId}/public/data/chats`; const chatQuery = query(collection(db, chatPath)); const unsubscribe = onSnapshot(chatQuery, (snapshot) => { const newMessages = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })).sort((a, b) => (a.timestamp?.toMillis() || 0) - (b.timestamp?.toMillis() || 0)); setMessages(newMessages); }, (error) => { console.error("Error fetching chat messages:", error); }); return () => unsubscribe(); }, []);
            useEffect(() => { messagesEndRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages]);
            if (!profile || profile?.role === 'visitor') { return e('div', {className:"p-6 md:p-8 text-center text-red-400"}, "Please login or sign up to participate in the chat."); }
            const handleSubmit = async (e) => { e.preventDefault(); const messageText = chatInput.trim(); if (!messageText) return; const chatData = { text: messageText, userId: profile?.uid, displayName: profile?.displayName, timestamp: serverTimestamp(), role: profile?.role }; try { const chatPath = `artifacts/${finalAppId}/public/data/chats`; await addDoc(collection(db, chatPath), chatData); setChatInput(''); } catch (error) { console.error("Error sending chat message:", error); } };
            const renderChatMessages = () => { return messages.map(msg => { const isMyMessage = msg.userId === profile?.uid; const timeString = msg.timestamp ? new Date(msg.timestamp.toMillis()).toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' }) : 'N/A'; const displaySender = msg.role === 'admin' ? `${msg.displayName} (Admin)` : (msg.displayName || `Customer: ${msg.userId?.substring(0, 8) + '...'}`); return e('div', {key:msg.id, className:`flex ${isMyMessage ? 'justify-end' : 'justify-start'}`}, e('div', {className:`max-w-xs md:max-w-md p-3 rounded-xl shadow-lg ${isMyMessage ? 'bg-indigo-600 text-white rounded-br-none shadow-indigo-500/30' : 'bg-gray-700 text-gray-200 rounded-tl-none shadow-md'}`}, e('div', {className:`font-semibold text-xs mb-1 ${isMyMessage ? 'text-indigo-200' : 'text-gray-400'}`}, isMyMessage ? 'You' : displaySender), e('p', {className:"break-words"}, msg.text), e('div', {className:`text-right text-xs mt-1 ${isMyMessage ? 'text-indigo-200' : 'text-gray-500'}`}, timeString))); }); };
            return e('div', {className:"flex flex-col h-[60vh] p-6 md:p-8"}, e('div', {className:"border-b border-gray-700 pb-4 mb-4"}, e('h2', {className:"text-xl font-bold text-indigo-500"}, "Buyer Chat (Real-time)"), e('p', {className:"text-sm text-gray-400"}, "Powered by Google Firestore. Your messages are public.")), e('div', {className:"flex-1 overflow-y-auto space-y-4 pr-2 custom-scrollbar"}, messages.length === 0 ? e('p', {className:"text-center text-gray-500 italic"}, "Start the conversation!") : renderChatMessages(), e('div', {ref:messagesEndRef})), e('form', {onSubmit:handleSubmit, className:"pt-4 border-t border-gray-700 mt-4"}, e('div', {className:"flex space-x-2"}, e('input', {type:"text", value:chatInput, onChange:(e) => setChatInput(e.target.value), placeholder:"Type your message...", required:true, className:"flex-1 p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"}), e('button', {type:"submit", className:"py-2 px-4 bg-indigo-600 text-white font-semibold rounded-lg shadow-lg shadow-indigo-500/40 hover:bg-indigo-700 transition duration-300"}, "Send"))));
        }
        const AuthViewComponent = () => { 
            const { login, signup, loading: authLoading } = useAuth();
            const [isLogin, setIsLogin] = useState(true);
            const [email, setEmail] = useState('');
            const [password, setPassword] = useState('');
            const [error, setError] = useState('');
            const [loading, setLoading] = useState(false);
            const handleSubmit = async (e) => { e.preventDefault(); setError(''); setLoading(true); try { if (isLogin) { await login(email, password); } else { await signup(email, password); } } catch (err) { console.error("Auth Error:", err); let message = "An unknown error occurred."; if (err.code.includes('auth/invalid-email')) message = "Invalid email format."; if (err.code.includes('auth/wrong-password')) message = "Invalid password."; if (err.code.includes('auth/user-not-found')) message = "No user found with this email."; if (err.code.includes('auth/email-already-in-use')) message = "This email is already registered."; if (err.code.includes('auth/weak-password')) message = "Password should be at least 6 characters."; setError(message); } finally { setLoading(false); } };
            return e('div', {className:"p-6 md:p-8 flex justify-center items-center h-[60vh]"}, e('div', {className:"bg-gray-800 p-8 rounded-xl shadow-2xl w-full max-w-md"}, e('h2', {className:"text-2xl font-bold text-indigo-500 mb-6 text-center"}, isLogin ? 'Login to SK. HSE.' : 'Sign Up for SK. HSE.'), e('div', {className:"flex mb-6"}, e('button', {onClick:() => setIsLogin(true), className:`flex-1 py-2 rounded-l-lg font-semibold transition duration-200 ${isLogin ? 'bg-indigo-600 text-white' : 'bg-gray-700 text-gray-400 hover:bg-gray-600'}`}, "Login"), e('button', {onClick:() => setIsLogin(false), className:`flex-1 py-2 rounded-r-lg font-semibold transition duration-200 ${!isLogin ? 'bg-indigo-600 text-white' : 'bg-gray-700 text-gray-400 hover:bg-gray-600'}`}, "Sign Up")), e('form', {onSubmit:handleSubmit}, e('div', {className:"mb-4"}, e('label', {className:"block text-sm font-medium text-gray-300 mb-1"}, "Email"), e('input', {type:"email", value:email, onChange:(e) => setEmail(e.target.value), required:true, className:"w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), e('div', {className:"mb-6"}, e('label', {className:"block text-sm font-medium text-gray-300 mb-1"}, "Password"), e('input', {type:"password", value:password, onChange:(e) => setPassword(e.target.value), required:true, className:"w-full p-3 bg-gray-700 border border-gray-600 rounded-lg text-white focus:ring-indigo-500 focus:border-indigo-500 shadow-inner"})), error && e('div', {className:"mb-4 p-3 bg-red-800 text-red-200 rounded-lg text-sm font-medium"}, error), e('button', {type:"submit", disabled:loading, className:"w-full py-3 bg-indigo-600 text-white font-semibold rounded-lg shadow-lg shadow-indigo-500/40 hover:bg-indigo-700 transition duration-300 disabled:opacity-50 flex items-center justify-center"}, loading ? e('div', {className:"animate-spin rounded-full h-5 w-5 border-b-2 border-white"}) : (isLogin ? 'Login' : 'Sign Up'))), e('p', {className:"text-sm text-gray-500 mt-6 text-center"}, "Admin login uses email: ", e('span', {className:"font-mono text-indigo-400"}, ADMIN_EMAIL))));
        }


        // --- Main Router & App Wrapper ---

        const AppContent = () => {
            const { user, profile, loading, logout } = useAuth();
            const [currentView, setCurrentView] = useState('home');
            const [isMobileView, setIsMobileView] = useState(false); // State for View Toggle

            const allTabs = [
                { id: 'admin', label: 'Admin Dashboard', roles: ['admin'] },
                { id: 'home', label: 'Home', roles: ['admin', 'customer', 'visitor'] },
                { id: 'about', label: 'About & Contact', roles: ['admin', 'customer', 'visitor'] },
                { id: 'products', label: 'Products & Pricing', roles: ['admin', 'customer', 'visitor'] },
                { id: 'order', label: 'Place Order', roles: ['admin', 'customer'] },
                { id: 'feedback', label: 'Give Feedback', roles: ['admin', 'customer'] },
                { id: 'chat', label: 'Buyer Chat', roles: ['admin', 'customer'] },
            ];

            const availableTabs = allTabs.filter(tab => tab.roles.includes(profile?.role)).slice(0, 5);

            const renderCurrentView = () => {
                if (!user && currentView !== 'auth') return e(AuthViewComponent);
                switch (currentView) {
                    case 'admin': return e(AdminDashboardComponent);
                    case 'about': return e(AboutViewComponent);
                    case 'products': return e(ProductsViewComponent);
                    case 'order': return e(OrderViewComponent);
                    case 'feedback': return e(FeedbackViewComponent);
                    case 'chat': return e(ChatViewComponent);
                    case 'auth': return e(AuthViewComponent);
                    case 'home': default: return e(HomeViewComponent);
                }
            };

            if (loading) {
                return e('div', {className:"h-screen flex justify-center items-center bg-gray-900"}, e(LoadingSpinner));
            }

            return e('div', {className:`mx-auto bg-gray-900 min-h-screen transition-all duration-500 ${isMobileView ? 'w-full max-w-sm' : 'max-w-6xl'}`},
                e('header', {className:"shadow-2xl shadow-gray-900/50"}, 
                    e('div', {className:"bg-gray-800 p-2 text-xs flex justify-between items-center text-gray-400 border-b border-gray-700 px-4"}, 
                        e('p', null, "Authenticated User ID: ", profile?.uid || 'N/A'),
                        e('div', {className:"flex items-center space-x-4"}, 
                            e('button', {onClick:() => setIsMobileView(prev => !prev), className:"font-semibold text-indigo-400 hover:text-indigo-300"}, "View Toggle: ", isMobileView ? 'Mobile' : 'Desktop'),
                            user ? e('button', {onClick:logout, className:"font-semibold text-red-400 hover:text-red-300"}, "Sign Out") : e('button', {onClick:() => setCurrentView('auth'), className:"font-semibold text-indigo-400 hover:text-indigo-300"}, "Login / Sign Up")
                        )
                    ),
                    e('div', {className:"bg-gray-900 p-6 flex items-center justify-between"}, 
                        e('div', {className:"flex items-center space-x-3"}, 
                            e('img', {src:"https://image.pollinations.ai/prompt/simple-mountain-logo-with-curve-and-ring-minimalist-stylized?width=50&height=50&seed=6662b0b0", alt:"SK. HSE. Logo", className:"w-10 h-10 object-contain filter-indigo"}),
                            e('h1', {className:"text-3xl font-extrabold text-indigo-500"}, "HSE.Pk")
                        ),
                        e('div', {className:"text-right"}, 
                            e('p', {className:"text-xl font-semibold text-white"}, profile?.displayName),
                            e('p', {className:`text-sm ${profile?.role === 'admin' ? 'text-indigo-400' : 'text-gray-400'}`}, profile?.role || 'Visitor')
                        )
                    ),
                    e('nav', {className:"flex bg-gray-800 border-t border-gray-700 overflow-x-auto"}, 
                        availableTabs.map(tab => e('button', {key:tab.id, onClick:() => setCurrentView(tab.id), className:`px-4 py-3 text-sm font-semibold whitespace-nowrap transition duration-300 ${currentView === tab.id ? 'bg-indigo-600 text-white shadow-inner shadow-indigo-900' : 'text-gray-300 hover:bg-gray-700 hover:text-white'}`}, tab.label))
                    )
                ),
                e('main', {className:"p-0"}, 
                    e(Suspense, {fallback:e('div', {className:"p-10 bg-gray-900"}, e(LoadingSpinner))}, renderCurrentView())
                ),
                e('style', {dangerouslySetInnerHTML:{__html:`
                    /* Global Styles */
                    body {
                        font-family: 'Inter', sans-serif;
                        background-color: #0d1117;
                    }
                    .filter-indigo {
                        filter: invert(40%) sepia(35%) saturate(300%) hue-rotate(220deg) brightness(120%) contrast(100%);
                    }
                    .text-shadow-indigo {
                        text-shadow: 0 0 5px rgba(99, 102, 241, 0.5), 0 0 10px rgba(99, 102, 241, 0.3);
                    }
                    .font-caveat {
                        font-family: 'Caveat', cursive;
                    }
                    .glowing-border-bottom {
                        border-bottom: 2px solid transparent;
                        border-image: linear-gradient(to right, #6366f1, #1f2937);
                        border-image-slice: 1;
                        padding-bottom: 4px;
                    }
                    .custom-scrollbar::-webkit-scrollbar {
                        width: 6px;
                    }
                    .custom-scrollbar::-webkit-scrollbar-thumb {
                        background-color: #4f46e5;
                        border-radius: 3px;
                    }
                    .custom-scrollbar::-webkit-scrollbar-track {
                        background: #1f2937;
                    }
                `}}),
            );
        };

        // --- App Root ---
        const App = () => e(AuthProvider, null, e(AppContent));

        // Start the application rendering
        document.addEventListener('DOMContentLoaded', () => {
            const rootElement = document.getElementById('root');
            if (rootElement) {
                createRoot(rootElement).render(e(App));
            }
        });

    </script>
</head>
<body>
    <div id="root"></div>
</body>
</html>


