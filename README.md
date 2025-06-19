import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, doc, updateDoc, deleteDoc } from 'firebase/firestore';

function App() {
  // Firebase configuration and initialization states
  const [db, setDb] = useState(null); // Firestore database instance
  const [auth, setAuth] = useState(null); // Firebase Auth instance
  const [userId, setUserId] = useState(null); // Current user's ID
  const [isAuthReady, setIsAuthReady] = useState(false); // Flag to check if authentication is ready

  // Employee data states for form input
  const [employees, setEmployees] = useState([]); // List of employees fetched from Firestore
  const [name, setName] = useState(''); // State for employee name input
  const [productionCapacity, setProductionCapacity] = useState(''); // State for production capacity input
  const [kpiThreshold, setKpiThreshold] = useState(100); // Default KPI threshold, can be adjusted by user

  // State for displaying messages (success or error) to the user
  const [message, setMessage] = useState({ text: '', type: '' });

  // useEffect hook for Firebase initialization and authentication listener
  // This runs once when the component mounts
  useEffect(() => {
    try {
      // Safely parse firebase config, provided by the Canvas environment
      const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
      const app = initializeApp(firebaseConfig); // Initialize Firebase app
      const firestore = getFirestore(app); // Get Firestore instance
      const authentication = getAuth(app); // Get Auth instance

      setDb(firestore); // Set db state
      setAuth(authentication); // Set auth state

      // Attempt to sign in with a custom token provided by the environment,
      // or fall back to anonymous sign-in if no token is available.
      const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
      if (initialAuthToken) {
        signInWithCustomToken(authentication, initialAuthToken)
          .then(() => {
            console.log("Signed in with custom token successfully.");
          })
          .catch((error) => {
            console.error("Error signing in with custom token:", error);
            signInAnonymously(authentication); // Fallback: sign in anonymously on error
          });
      } else {
        signInAnonymously(authentication); // Sign in anonymously if no custom token
      }

      // Set up an authentication state change listener.
      // This ensures we know when the user is authenticated and ready to interact with Firestore.
      const unsubscribeAuth = onAuthStateChanged(authentication, (user) => {
        if (user) {
          setUserId(user.uid); // Set userId if authenticated
          setIsAuthReady(true); // Mark auth as ready
        } else {
          setUserId(crypto.randomUUID()); // Generate a random ID for anonymous users if no user is found
          setIsAuthReady(true); // Mark auth as ready even for anonymous users
        }
      });

      // Cleanup function to unsubscribe from the auth listener when the component unmounts
      return () => unsubscribeAuth();
    } catch (error) {
      console.error("Failed to initialize Firebase:", error);
      setMessage({ text: "Lỗi: Không thể khởi tạo Firebase. Vui lòng thử lại.", type: "error" });
    }
  }, []); // Empty dependency array ensures this effect runs only once on mount

  // useEffect hook for fetching and real-time listening to employee data from Firestore
  // This runs when db, userId, or isAuthReady states change
  useEffect(() => {
    // Ensure Firebase instances and auth state are ready before attempting to fetch data
    if (db && userId && isAuthReady) {
      // Get the app ID, provided by the Canvas environment, or use a default
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
      // Reference to the 'employees' collection within the public data path in Firestore
      // This path ensures data is accessible according to Firebase security rules for public data.
      const employeesCollectionRef = collection(db, `artifacts/${appId}/public/data/employees`);

      // Set up a real-time listener (onSnapshot) to the employees collection.
      // Any changes to the collection in Firestore will automatically update the 'employees' state.
      const unsubscribe = onSnapshot(employeesCollectionRef, (snapshot) => {
        const employeesData = snapshot.docs.map(doc => ({
          id: doc.id, // Document ID
          ...doc.data() // All other fields from the document
        }));
        setEmployees(employeesData); // Update the employees state
      }, (error) => {
        console.error("Error fetching employees:", error);
        setMessage({ text: "Lỗi khi tải dữ liệu nhân viên.", type: "error" });
      });

      // Cleanup function to unsubscribe from the Firestore listener when the component unmounts or dependencies change
      return () => unsubscribe();
    }
  }, [db, userId, isAuthReady]); // Dependencies for this effect

  // Function to calculate KPI based on production capacity and threshold
  const calculateKPI = (capacity) => {
    const parsedCapacity = parseFloat(capacity); // Convert capacity to a floating-point number
    if (isNaN(parsedCapacity)) return 'N/A'; // Return 'N/A' if capacity is not a valid number
    return parsedCapacity >= kpiThreshold ? 'Đạt' : 'Chưa đạt'; // Check if capacity meets or exceeds the threshold
  };

  // Handler for adding a new employee or updating an existing one (current implementation is add only)
  const handleSubmit = async (e) => {
    e.preventDefault(); // Prevent default form submission behavior

    // Basic input validation
    if (!name || !productionCapacity) {
      setMessage({ text: "Vui lòng nhập đầy đủ tên và năng lực sản xuất.", type: "error" });
      return;
    }

    // Ensure database instance is available
    if (!db) {
      setMessage({ text: "Lỗi: Cơ sở dữ liệu chưa sẵn sàng.", type: "error" });
      return;
    }

    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const employeesCollectionRef = collection(db, `artifacts/${appId}/public/data/employees`);

    try {
      // Add a new document (employee record) to the Firestore collection
      await addDoc(employeesCollectionRef, {
        name: name, // Employee name
        productionCapacity: parseFloat(productionCapacity), // Production capacity (converted to number)
        kpi: calculateKPI(productionCapacity), // Calculated KPI status
        approved: false, // Default approval status is false
        timestamp: new Date(), // Timestamp of creation
        createdBy: userId // ID of the user who added this record
      });
      setMessage({ text: "Đã thêm nhân viên thành công!", type: "success" }); // Show success message
      setName(''); // Clear name input
      setProductionCapacity(''); // Clear production capacity input
    } catch (error) {
      console.error("Error adding document: ", error);
      setMessage({ text: `Lỗi khi thêm nhân viên: ${error.message}`, type: "error" }); // Show error message
    }
  };

  // Function to toggle the approval status of an employee
  const toggleApproval = async (employeeId, currentStatus) => {
    if (!db) return; // Ensure db is initialized

    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    // Reference to the specific employee document by its ID
    const employeeDocRef = doc(db, `artifacts/${appId}/public/data/employees`, employeeId);
    try {
      // Update the 'approved' field to the opposite of its current status
      await updateDoc(employeeDocRef, {
        approved: !currentStatus
      });
      setMessage({ text: "Đã cập nhật trạng thái duyệt!", type: "success" }); // Show success message
    } catch (error) {
      console.error("Error updating approval status: ", error);
      setMessage({ text: `Lỗi khi cập nhật trạng thái duyệt: ${error.message}`, type: "error" }); // Show error message
    }
  };

  // Function to delete an employee record
  const deleteEmployee = async (employeeId) => {
    if (!db) return; // Ensure db is initialized

    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    // Reference to the specific employee document by its ID
    const employeeDocRef = doc(db, `artifacts/${appId}/public/data/employees`, employeeId);
    try {
      // Delete the document from Firestore
      await deleteDoc(employeeDocRef);
      setMessage({ text: "Đã xóa nhân viên thành công!", type: "success" }); // Show success message
    } catch (error) {
      console.error("Error deleting employee: ", error);
      setMessage({ text: `Lỗi khi xóa nhân viên: ${error.message}`, type: "error" }); // Show error message
    }
  };

  return (
    // Main container for the application, centered and responsive
    <div className="min-h-screen bg-gray-100 flex flex-col items-center p-4 font-sans">
      {/* Viewport meta tag for responsive design */}
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      {/* Tailwind CSS CDN for styling */}
      <script src="https://cdn.tailwindcss.com"></script>
      {/* Custom styles for font and scrollbar */}
      <style>
        {`
          @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
          body {
            font-family: 'Inter', sans-serif; /* Apply Inter font globally */
          }
          /* Custom scrollbar for better aesthetics */
          ::-webkit-scrollbar {
            width: 8px; /* Width of the scrollbar */
          }
          ::-webkit-scrollbar-track {
            background: #f1f1f1; /* Background of the scrollbar track */
            border-radius: 10px; /* Rounded corners for the track */
          }
          ::-webkit-scrollbar-thumb {
            background: #888; /* Color of the scrollbar thumb */
            border-radius: 10px; /* Rounded corners for the thumb */
          }
          ::-webkit-scrollbar-thumb:hover {
            background: #555; /* Color of the thumb on hover */
          }
        `}
      </style>

      {/* Message Box for displaying success/error messages */}
      {message.text && (
        <div className={`p-3 rounded-lg shadow-md mb-4 text-center w-full max-w-md ${
          message.type === 'success' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
        }`}>
          {message.text}
        </div>
      )}

      {/* Main content card */}
      <div className="bg-white p-6 rounded-xl shadow-lg w-full max-w-2xl mb-8">
        <h1 className="text-3xl font-bold text-gray-800 mb-6 text-center">Quản Lý Năng Lực & KPI Nhân Viên</h1>
        {/* Display current user ID for collaborative context */}
        <p className="text-sm text-gray-600 mb-4 text-center">
            ID Người Dùng của bạn: <span className="font-semibold text-blue-600 break-words">{userId || 'Đang tải...'}</span>
        </p>

        {/* KPI Threshold Input Section */}
        <div className="mb-6">
          <label htmlFor="kpiThreshold" className="block text-gray-700 text-sm font-semibold mb-2">
            Ngưỡng KPI (ví dụ: 100 sản phẩm/tháng):
          </label>
          <input
            type="number"
            id="kpiThreshold"
            className="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200"
            value={kpiThreshold}
            onChange={(e) => setKpiThreshold(parseFloat(e.target.value))} // Update kpiThreshold state
            placeholder="Nhập ngưỡng KPI"
          />
        </div>

        {/* Employee Input Form Section */}
        <form onSubmit={handleSubmit} className="mb-8 p-4 border border-gray-200 rounded-lg shadow-inner">
          <h2 className="text-xl font-semibold text-gray-700 mb-4">Thêm Nhân Viên Mới</h2>
          <div className="mb-4">
            <label htmlFor="name" className="block text-gray-700 text-sm font-semibold mb-2">
              Tên Nhân Viên:
            </label>
            <input
              type="text"
              id="name"
              className="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200"
              value={name}
              onChange={(e) => setName(e.target.value)} // Update name state
              placeholder="Nhập tên nhân viên"
              required // Make input required
            />
          </div>
          <div className="mb-4">
            <label htmlFor="productionCapacity" className="block text-gray-700 text-sm font-semibold mb-2">
              Năng Lực Sản Xuất (số lượng):
            </label>
            <input
              type="number"
              id="productionCapacity"
              className="shadow-sm appearance-none border rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition duration-200"
              value={productionCapacity}
              onChange={(e) => setProductionCapacity(e.target.value)} // Update productionCapacity state
              placeholder="Nhập năng lực sản xuất (ví dụ: 120)"
              required // Make input required
            />
          </div>
          <button
            type="submit"
            className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-75 transition duration-200 transform hover:scale-105"
          >
            Thêm Nhân Viên
          </button>
        </form>

        {/* Employee List Section */}
        <h2 className="text-xl font-semibold text-gray-700 mb-4">Danh Sách Nhân Viên</h2>
        {employees.length === 0 ? (
          // Display message if no employees are present
          <p className="text-gray-500 text-center">Chưa có nhân viên nào. Hãy thêm một người!</p>
        ) : (
          // Table to display employee data
          <div className="overflow-x-auto rounded-lg shadow-md border border-gray-200">
            <table className="min-w-full divide-y divide-gray-200">
              <thead className="bg-gray-50">
                <tr>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Tên
                  </th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Năng Lực Sản Xuất
                  </th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    KPI
                  </th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Duyệt
                  </th>
                  <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Hành Động
                  </th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {employees.map((employee) => (
                  <tr key={employee.id} className="hover:bg-gray-50">
                    <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">
                      {employee.name}
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                      {employee.productionCapacity}
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-sm">
                      {/* Display KPI status with dynamic styling */}
                      <span className={`px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${
                        calculateKPI(employee.productionCapacity) === 'Đạt' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
                      }`}>
                        {calculateKPI(employee.productionCapacity)}
                      </span>
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-sm">
                      {/* Button to toggle approval status */}
                      <button
                        onClick={() => toggleApproval(employee.id, employee.approved)}
                        className={`py-1 px-3 rounded-full text-white text-xs font-medium focus:outline-none focus:ring-2 focus:ring-opacity-75 transition duration-200 ${
                          employee.approved ? 'bg-green-500 hover:bg-green-600 focus:ring-green-400' : 'bg-yellow-500 hover:bg-yellow-600 focus:ring-yellow-400'
                        }`}
                      >
                        {employee.approved ? 'Đã Duyệt' : 'Chưa Duyệt'}
                      </button>
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                      {/* Button to delete employee record */}
                      <button
                        onClick={() => deleteEmployee(employee.id)}
                        className="text-red-600 hover:text-red-900 ml-4 focus:outline-none focus:ring-2 focus:ring-red-500 focus:ring-opacity-75 transition duration-200"
                      >
                        Xóa
                      </button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </div>
    </div>
  );
}

export default App; // Export the App component as default
