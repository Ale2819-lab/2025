import React, { useState, useEffect, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, query, onSnapshot, serverTimestamp } from 'firebase/firestore';

// Global variables provided by the Canvas environment
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

function App() {
  const [selectedFiles, setSelectedFiles] = useState([]);
  const [uploading, setUploading] = useState(false);
  const [uploadProgress, setUploadProgress] = useState(0);
  const [message, setMessage] = useState('');
  const [uploadedFileMetadata, setUploadedFileMetadata] = useState([]);
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);

  // Initialize Firebase and handle authentication
  useEffect(() => {
    try {
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const firebaseAuth = getAuth(app);
      setDb(firestore);
      setAuth(firebaseAuth);

      onAuthStateChanged(firebaseAuth, async (user) => {
        if (user) {
          setUserId(user.uid);
        } else {
          // Sign in anonymously if no custom token is provided or if it fails
          if (initialAuthToken) {
            try {
              await signInWithCustomToken(firebaseAuth, initialAuthToken);
              setUserId(firebaseAuth.currentUser.uid);
            } catch (error) {
              console.error("Error signing in with custom token, signing in anonymously:", error);
              await signInAnonymously(firebaseAuth);
              setUserId(firebaseAuth.currentUser.uid);
            }
          } else {
            await signInAnonymously(firebaseAuth);
            setUserId(firebaseAuth.currentUser.uid);
          }
        }
        setIsAuthReady(true); // Auth state has been checked
      });
    } catch (error) {
      setMessage(`Error initializing Firebase: ${error.message}`);
      console.error("Firebase initialization error:", error);
    }
  }, []);

  // Listen for file metadata changes in Firestore
  useEffect(() => {
    if (!db || !isAuthReady) return;

    // Define collection path based on public data security rules
    // For shared/public data: /artifacts/{appId}/public/data/{your_collection_name}
    const filesCollectionRef = collection(db, `artifacts/${appId}/public/data/uploaded_files`);
    const q = query(filesCollectionRef);

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const files = [];
      snapshot.forEach((doc) => {
        files.push({ id: doc.id, ...doc.data() });
      });
      // Sort files by upload time in descending order
      files.sort((a, b) => (b.uploadedAt?.toDate() || 0) - (a.uploadedAt?.toDate() || 0));
      setUploadedFileMetadata(files);
    }, (error) => {
      setMessage(`Error fetching uploaded file metadata: ${error.message}`);
      console.error("Firestore snapshot error:", error);
    });

    return () => unsubscribe();
  }, [db, isAuthReady]);

  // Handle file selection
  const handleFileChange = useCallback((event) => {
    setMessage(''); // Clear previous messages
    const files = Array.from(event.target.files);
    setSelectedFiles(files);
  }, []);

  // Handle drag over for drag-and-drop
  const handleDragOver = useCallback((event) => {
    event.preventDefault();
    event.stopPropagation();
    event.dataTransfer.dropEffect = 'copy'; // Visual feedback for copying
  }, []);

  // Handle drop for drag-and-drop
  const handleDrop = useCallback((event) => {
    event.preventDefault();
    event.stopPropagation();
    setMessage(''); // Clear previous messages
    const files = Array.from(event.dataTransfer.files);
    setSelectedFiles(files);
  }, []);

  // Simulate file upload and save metadata to Firestore
  const simulateUpload = useCallback(async () => {
    if (selectedFiles.length === 0) {
      setMessage('Please select files to upload.');
      return;
    }
    if (!db || !userId) {
      setMessage('Firebase not initialized or user not authenticated. Please wait.');
      return;
    }

    setUploading(true);
    setUploadProgress(0);
    setMessage('Simulating upload...');

    for (const file of selectedFiles) {
      const fileName = file.name;
      const fileSize = file.size;
      const fileType = file.type;

      // Simulate progress for each file
      let currentProgress = 0;
      const interval = setInterval(() => {
        currentProgress += 10;
        if (currentProgress <= 100) {
          setUploadProgress(currentProgress);
        } else {
          clearInterval(interval);
        }
      }, 100);

      try {
        // Generate a unique ID for the document
        const docId = `${userId}_${Date.now()}_${Math.random().toString(36).substring(2, 9)}`;
        const fileDocRef = doc(db, `artifacts/${appId}/public/data/uploaded_files`, docId);

        // Save file metadata to Firestore (simulating storage)
        await setDoc(fileDocRef, {
          fileName: fileName,
          fileSize: fileSize, // in bytes
          fileType: fileType,
          uploadedBy: userId, // Store the ID of the uploader
          uploadedAt: serverTimestamp(), // Firestore timestamp
          // In a real app, you'd save a reference to the actual file in cloud storage here
          // For simulation, we'll just generate a conceptual shareable link.
          shareableLink: `https://your-domain.com/uploads/${docId}` // Conceptual link
        });

        clearInterval(interval); // Ensure interval is cleared
        setUploadProgress(100);
        setMessage(`'${fileName}' upload simulated successfully! Metadata saved.`);

      } catch (error) {
        clearInterval(interval); // Ensure interval is cleared on error
        setMessage(`Error simulating upload for '${fileName}': ${error.message}`);
        console.error("Firestore upload error:", error);
      }
    }

    setUploading(false);
    setSelectedFiles([]); // Clear selected files after simulation
  }, [selectedFiles, db, userId]);

  return (
    <div className="min-h-screen bg-gray-100 flex items-center justify-center p-4">
      <div className="bg-white rounded-lg shadow-xl p-8 w-full max-w-2xl">
        <h1 className="text-3xl font-extrabold text-gray-900 mb-6 text-center">
          Web File Upload Portal
        </h1>
        <p className="text-sm text-gray-600 mb-8 text-center">
          <strong className="text-red-500">Disclaimer:</strong> This is a client-side simulation. Files are NOT actually stored on a server. Only file metadata is saved to Firestore for demonstration purposes.
        </p>

        {!isAuthReady ? (
          <p className="text-center text-blue-600 font-medium">Initializing Firebase and authenticating...</p>
        ) : (
          <p className="text-center text-gray-700 mb-4">
            {userId ? `Logged in as: ${userId}` : 'Authentication failed or pending.'}
          </p>
        )}

        <div
          className="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center cursor-pointer hover:border-blue-500 transition-all duration-300 mb-6"
          onDragOver={handleDragOver}
          onDrop={handleDrop}
          onClick={() => document.getElementById('fileInput').click()}
        >
          <input
            type="file"
            id="fileInput"
            multiple
            className="hidden"
            onChange={handleFileChange}
            disabled={uploading}
          />
          {selectedFiles.length > 0 ? (
            <p className="text-lg text-gray-700">
              {selectedFiles.length} file(s) selected: {selectedFiles.map(f => f.name).join(', ')}
            </p>
          ) : (
            <p className="text-lg text-gray-500">
              Drag & drop files here, or <span className="text-blue-600 font-semibold">click to browse</span>
            </p>
          )}
          <p className="text-sm text-gray-400 mt-2">(Max 100MB per file for real-world practical limits, but this is a simulation)</p>
        </div>

        <button
          onClick={simulateUpload}
          className={`w-full py-3 px-4 rounded-lg text-white font-semibold transition-all duration-300 ${
            selectedFiles.length === 0 || uploading
              ? 'bg-blue-300 cursor-not-allowed'
              : 'bg-blue-600 hover:bg-blue-700 shadow-md'
          }`}
          disabled={selectedFiles.length === 0 || uploading || !isAuthReady}
        >
          {uploading ? `Uploading... ${uploadProgress}%` : 'Simulate Upload'}
        </button>

        {uploading && (
          <div className="w-full bg-gray-200 rounded-full h-2.5 mt-4">
            <div
              className="bg-blue-600 h-2.5 rounded-full transition-all duration-300"
              style={{ width: `${uploadProgress}%` }}
            ></div>
          </div>
        )}

        {message && (
          <p className="mt-4 text-center text-sm font-medium text-gray-700">{message}</p>
        )}

        <div className="mt-8 border-t border-gray-200 pt-6">
          <h2 className="text-2xl font-bold text-gray-900 mb-4 text-center">
            "Uploaded" File Metadata (Public)
          </h2>
          {uploadedFileMetadata.length === 0 ? (
            <p className="text-center text-gray-500">No files uploaded yet.</p>
          ) : (
            <ul className="space-y-3">
              {uploadedFileMetadata.map((file) => (
                <li
                  key={file.id}
                  className="bg-gray-50 p-4 rounded-md flex items-center justify-between shadow-sm"
                >
                  <div>
                    <p className="text-md font-semibold text-gray-800 break-words">
                      {file.fileName}
                    </p>
                    <p className="text-sm text-gray-600">
                      Size: {(file.fileSize / (1024 * 1024)).toFixed(2)} MB | Type: {file.fileType}
                    </p>
                    <p className="text-xs text-gray-500">
                      Uploaded by: {file.uploadedBy === userId ? 'You' : file.uploadedBy}
                      {file.uploadedAt && ` on ${new Date(file.uploadedAt.toDate()).toLocaleString()}`}
                    </p>
                  </div>
                  {file.shareableLink && (
                    <a
                      href={file.shareableLink}
                      target="_blank"
                      rel="noopener noreferrer"
                      className="text-blue-500 hover:underline text-sm ml-4 whitespace-nowrap"
                    >
                      Share Link
                    </a>
                  )}
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>
    </div>
  );
}

export default App;
