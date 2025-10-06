# SmartXpend
Finance Tracker IOS App
//
//  SmartExpend.swift
//  SmartExpend App - Sample UI Layout
//
//  This file contains a complete, single-file SwiftUI implementation of the SmartExpend
//  app layout. It includes all the necessary views, data models, and a TabView for
//  bottom navigation.
//
//  This version has been updated to use local in-memory authentication and
//  data storage, enabling demonstration without Firebase. It also
//  incorporates new UI features like interactive financial cards, a category filter,
//  and an interactive pie chart.
//
//

import SwiftUI
import Charts
import LocalAuthentication
import Combine
import UserNotifications // Added for local notifications

// MARK: - Data Models

/// A simple data model for the financial cards.
struct FinancialData: Identifiable {
    let id = UUID()
    let title: String
    let value: Double
    let iconName: String
    let color: Color
    let prefix: String?
}

/// A data model for the expense table and charts.
struct Expense: Identifiable, Codable {
    var id = UUID()
    var category: String
    var amount: Double
    var date: Date = Date()
}

/// A data model for the line chart.
struct FinancialTrend: Identifiable {
    let id = UUID()
    let month: String
    let expenses: Double
    let income: Double
}

/// A data model for the user's account details.
struct User: Identifiable, Codable {
    var id: UUID = UUID()
    var name: String
    var age: String
    var gender: String
    var maritalStatus: String
    var number: String
    var email: String
    var monthlySalary: Double
    var expenses: [Expense] = []
}

/// A data model for the pie chart slices.
struct PieSlice: Identifiable, Equatable {
    let id = UUID()
    let category: String
    let amount: Double
    let color: Color
}

// MARK: - Notification Manager for Local Notifications

/// Manages notification authorization and scheduling local notifications.
class NotificationManager {
    /// Requests authorization from the user for sending notifications.
    static func requestAuthorization() {
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge]) { granted, error in
            if let error = error {
                print("Notification authorization error: \(error)")
            }
        }
    }

    /// Schedules a notification to alert user of budget overage.
    static func scheduleBudgetOverageNotification() {
        let content = UNMutableNotificationContent()
        content.title = "Budget Alert"
        content.body = "You've exceeded your monthly budget. Review your spending to stay on track."
        content.sound = .default
        let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 2, repeats: false)
        let request = UNNotificationRequest(identifier: "budgetOverage", content: content, trigger: trigger)
        UNUserNotificationCenter.current().add(request) { error in
            if let error = error {
                print("Failed to schedule budget overage notification: \(error)")
            }
        }
    }
    // Extend: Add methods for bills/goals
}

// MARK: - App State

/// This class holds the app's global state, including the currently authenticated user.
/// Uses local in-memory storage for demonstration purposes.
class AppState: ObservableObject {
    @Published var currentUser: User?
    @Published var isAuthReady = false // Will be true after loading saved data
    
    // Local in-memory store of users for demo purposes
    @Published private(set) var users: [User] = []
    
    private let usersKey = "SmartExpendUsersKey"
    private let currentUserIdKey = "SmartExpendCurrentUserIdKey"
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        loadUsers()
    }
    
    // MARK: - Persistence
    
    private func saveUsers() {
        do {
            let data = try JSONEncoder().encode(users)
            UserDefaults.standard.set(data, forKey: usersKey)
        } catch {
            print("Failed to save users:", error)
        }
    }
    
    func saveCurrentUserId() {
        if let currentUser = currentUser {
            UserDefaults.standard.set(currentUser.id.uuidString, forKey: currentUserIdKey)
        } else {
            UserDefaults.standard.removeObject(forKey: currentUserIdKey)
        }
    }
    
    private func loadUsers() {
        DispatchQueue.global(qos: .userInitiated).async {
            var loadedUsers: [User] = []
            if let data = UserDefaults.standard.data(forKey: self.usersKey) {
                do {
                    loadedUsers = try JSONDecoder().decode([User].self, from: data)
                } catch {
                    print("Failed to load users:", error)
                }
            }
            let savedUserIdString = UserDefaults.standard.string(forKey: self.currentUserIdKey)
            let savedUserId = savedUserIdString != nil ? UUID(uuidString: savedUserIdString!) : nil
            
            DispatchQueue.main.async {
                self.users = loadedUsers
                if let savedId = savedUserId {
                    self.currentUser = loadedUsers.first(where: { $0.id == savedId })
                }
                self.isAuthReady = true
            }
        }
    }
    
    private func updateUser(_ user: User) {
        if let idx = users.firstIndex(where: { $0.id == user.id }) {
            users[idx] = user
        } else {
            users.append(user)
        }
    }
    
    /// Updates the currentUser with the provided updated user data,
    /// updates the users array, and persists changes.
    func updateCurrentUser(_ updated: User) {
        currentUser = updated
        if let idx = users.firstIndex(where: { $0.id == updated.id }) {
            users[idx] = updated
        } else {
            users.append(updated)
        }
        saveUsers()
        saveCurrentUserId()
    }
    
    // MARK: - Computed properties for currentUser
    var totalExpenses: Double {
        currentUser?.expenses.reduce(0) { $0 + $1.amount } ?? 0
    }
    
    var totalIncome: Double {
        currentUser?.monthlySalary ?? 0
    }
    
    // MARK: - Authentication and User Management
    
    /// Registers a new user. Returns error message if failure occurs, nil on success.
    func createUser(name: String, age: String, gender: String, maritalStatus: String, number: String, email: String, password: String, monthlySalary: Double) -> String? {
        // Check if email already exists
        if users.contains(where: { $0.email.lowercased() == email.lowercased() }) {
            return "Email is already registered."
        }
        
        // For demo, password is not stored or verified securely
        // Create new user
        let newUser = User(name: name, age: gender, gender: maritalStatus, maritalStatus: maritalStatus, number: number, email: email, monthlySalary: monthlySalary)
        users.append(newUser)
        currentUser = newUser
        
        saveUsers()
        saveCurrentUserId()
        
        return nil
    }
    
    /// Signs in a user matching the email. Password is not verified for demo purposes.
    func signIn(email: String, password: String) -> String? {
        // Find user by email
        if let user = users.first(where: { $0.email.lowercased() == email.lowercased() }) {
            // Password check omitted for demo
            currentUser = user
            
            saveCurrentUserId()
            return nil
        } else {
            return "No user found with this email."
        }
    }
    
    /// Signs out the current user
    func signOut() {
        currentUser = nil
        
        saveCurrentUserId()
    }
    
    /// Adds an expense to the current user's expenses and updates the user in the users array
    func addExpense(_ expense: Expense) {
        guard var user = currentUser else { return }
        user.expenses.append(expense)
        currentUser = user
        // Update user in users array
        if let idx = users.firstIndex(where: { $0.id == user.id }) {
            users[idx] = user
        } else {
            users.append(user)
        }
        
        saveUsers()
        saveCurrentUserId()
        
        // Notification: Check budget overage after adding expense
        if totalExpenses > totalIncome {
            NotificationManager.scheduleBudgetOverageNotification()
        }
    }
}

// MARK: - App Main Structure

@main
struct SmartExpendApp: App {
    @StateObject private var appState = AppState()

    var body: some Scene {
        WindowGroup {
            Group {
                if appState.isAuthReady {
                    ContentView()
                        .environmentObject(appState)
                } else {
                    ProgressView()
                }
            }
            .onAppear {
                // Request notification authorization on app launch
                NotificationManager.requestAuthorization()
            }
        }
    }
}

// MARK: - App Root View

struct ContentView: View {
    @EnvironmentObject var appState: AppState
    
    var body: some View {
        if appState.currentUser != nil {
            MainTabView()
                .background(FinanceBackground())
                .ignoresSafeArea()
        } else {
            SplashView()
        }
    }
}

// MARK: - 1. Splash / Welcome Screen

struct SplashView: View {
    @State private var showingAuthView = false
    @State private var authInitialMode: AuthView.Mode = .signIn
    
    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottom) {
                Color.white
                    .ignoresSafeArea()
                
                VStack {
                    Spacer()
                    
                    Image(systemName: "banknote")
                        .font(.system(size: 80, weight: .bold))
                        .foregroundColor(.black)
                        .shadow(color: .black.opacity(0.25), radius: 14, y: 8)
                    
                    Text("SmartExpend")
                        .font(.system(size: 38, weight: .heavy))
                        .foregroundColor(.black)
                        .shadow(color: .black.opacity(0.1), radius: 5, x: 0, y: 1)
                        .padding(.top, 24)
                        .multilineTextAlignment(.center)
                        .frame(maxWidth: .infinity, alignment: .center)
                    
                    Text("Track. Save. Grow.")
                        .font(.system(size: 20, weight: .semibold))
                        .foregroundColor(Color(red: 0.62, green: 0.54, blue: 0.30))
                        .shadow(color: Color(red: 0.62, green: 0.54, blue: 0.30).opacity(0.5), radius: 4, x: 0, y: 1)
                        .padding(.top, 24)
                        .multilineTextAlignment(.center)
                        .frame(maxWidth: .infinity, alignment: .center)
                    
                    Spacer()
                    
                    VStack(spacing: 18) {
                        Button(action: {
                            authInitialMode = .signIn
                            showingAuthView = true
                        }) {
                            Text("Sign In")
                                .font(.headline.bold())
                                .foregroundColor(.white)
                                .frame(maxWidth: .infinity)
                                .padding(16)
                                .background(
                                    LinearGradient(
                                        colors: [Color(hex: "#A4508B"), Color(hex: "#5F0A87")],
                                        startPoint: .leading,
                                        endPoint: .trailing)
                                )
                                .cornerRadius(16)
                                .shadow(color: Color(hex: "#5F0A87").opacity(0.6), radius: 8, x: 0, y: 4)
                        }
                        
                        Button(action: {
                            authInitialMode = .signUp
                            showingAuthView = true
                        }) {
                            Text("Create Account")
                                .font(.headline.bold())
                                .foregroundColor(Color(hex: "#A4508B"))
                                .frame(maxWidth: .infinity)
                                .padding(16)
                                .background(Color.clear)
                                .overlay(
                                    RoundedRectangle(cornerRadius: 16)
                                        .stroke(Color(hex: "#A4508B"), lineWidth: 2)
                                )
                        }
                    }
                    .frame(maxWidth: 480)
                    .frame(maxWidth: .infinity, alignment: .center)
                    .padding(.horizontal, 20)
                    .padding(.bottom, 24)
                    
                    WavyBottomShape()
                        .fill(
                            LinearGradient(
                                gradient: Gradient(colors: [Color(hex: "#A4508B"), .black]),
                                startPoint: .topLeading,
                                endPoint: .bottomTrailing
                            )
                        )
                        .frame(height: 210)
                        .ignoresSafeArea(edges: .bottom)
                }
                .frame(maxWidth: .infinity, alignment: .center)
            }
            .navigationDestination(isPresented: $showingAuthView) {
                AuthView(initialMode: authInitialMode)
            }
        }
        .padding(.top, 40)
        .padding(.bottom, 40)
    }
}

struct WavyBottomShape: Shape {
    func path(in rect: CGRect) -> Path {
        var path = Path()
        path.move(to: .zero)
        path.addLine(to: CGPoint(x: 0, y: rect.height * 0.4))
        path.addCurve(to: CGPoint(x: rect.width, y: rect.height * 0.38),
                      control1: CGPoint(x: rect.width * 0.23, y: rect.height * 0.6),
                      control2: CGPoint(x: rect.width * 0.77, y: rect.height * 0.22))
        path.addLine(to: CGPoint(x: rect.width, y: 0))
        path.addLine(to: CGPoint(x: rect.width, y: rect.height))
        path.addLine(to: CGPoint(x: 0, y: rect.height))
        path.closeSubpath()
        return path
    }
}

// Adding Color extension for hex convenience
extension Color {
    init(hex: String) {
        let hex = hex.trimmingCharacters(in: CharacterSet.alphanumerics.inverted)
        var int: UInt64 = 0
        Scanner(string: hex).scanHexInt64(&int)
        let a, r, g, b: UInt64
        switch hex.count {
        case 3: // RGB (12-bit)
            (a, r, g, b) = (255, (int >> 8) * 17, (int >> 4 & 0xF) * 17, (int & 0xF) * 17)
        case 6: // RGB (24-bit)
            (a, r, g, b) = (255, int >> 16, int >> 8 & 0xFF, int & 0xFF)
        case 8: // ARGB (32-bit)
            (a, r, g, b) = (int >> 24, int >> 16 & 0xFF, int >> 8 & 0xFF, int & 0xFF)
        default:
            (a, r, g, b) = (255, 0, 0, 0)
        }
        self.init(
            .sRGB, 
            red: Double(r) / 255, 
            green: Double(g) / 255, 
            blue: Double(b) / 255, 
            opacity: Double(a) / 255
        )
    }
}

// MARK: - Unified AuthView (Sign In / Sign Up with social buttons)

struct AuthView: View {
    enum Mode {
        case signIn
        case signUp
    }
    
    @EnvironmentObject var appState: AppState
    @Environment(\.dismiss) var dismiss
    
    // Mode toggle
    @State private var mode: Mode
    
    // Shared Fields
    @State private var email = ""
    @State private var password = ""
    
    // Sign Up Fields
    @State private var confirmPassword = ""
    @State private var name = ""
    @State private var age = ""
    @State private var gender = ""
    @State private var maritalStatus = ""
    @State private var number = ""
    @State private var monthlySalaryInput = ""
    
    // Error handling
    @State private var showError = false
    @State private var errorMessage = ""
    
    // Alert for social login demo
    @State private var showingSocialAlert = false
    @State private var socialAlertMessage = ""
    
    // MARK: - Biometric Authentication State
    @State private var isAuthenticatingBiometric = false
    @State private var biometricError: String? = nil
    
    init(initialMode: Mode = .signIn) {
        _mode = State(initialValue: initialMode)
    }
    
    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                Picker("Authentication Mode", selection: $mode) {
                    Text("Sign In").tag(Mode.signIn)
                    Text("Create Account").tag(Mode.signUp)
                }
                .pickerStyle(SegmentedPickerStyle())
                .padding(.horizontal, 20)
                .tint(Color.financeSecondaryAccent)
                .frame(maxWidth: 480)
                .frame(maxWidth: .infinity, alignment: .center)
                
                if mode == .signIn {
                    signInView
                } else {
                    signUpView
                }
                
                Divider()
                    .padding(.horizontal, 20)
                
                socialLoginButtons
                
                Spacer(minLength: 20)
            }
            .padding(.top, 40)
            .padding(.bottom, 40)
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
        }
        .scrollBounceBehavior(.basedOnSize)
        .scrollDismissesKeyboard(.interactively)
        .background(FinanceBackground())
        .ignoresSafeArea()
        .navigationTitle(mode == .signIn ? "Sign In" : "Create Account")
        .navigationBarTitleDisplayMode(.inline)
        .alert(socialAlertMessage, isPresented: $showingSocialAlert) {
            Button("OK", role: .cancel) {}
        }
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    // MARK: - Sign In View
    private var signInView: some View {
        VStack(spacing: 16) {
            Group {
                TextField("Email", text: $email)
                    .keyboardType(.emailAddress)
                    .autocapitalization(.none)
                    .textContentType(.emailAddress)
                    .textFieldStyle(.roundedBorder)
                
                SecureField("Password", text: $password)
                    .textContentType(.password)
                    .textFieldStyle(.roundedBorder)
            }
            .padding(.horizontal, 20)
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
            
            if showError || biometricError != nil {
                Text(biometricError ?? (errorMessage.isEmpty ? "Invalid email or password." : errorMessage))
                    .foregroundColor(.red)
                    .padding(.horizontal, 20)
                    .multilineTextAlignment(.center)
                    .frame(maxWidth: 480)
                    .frame(maxWidth: .infinity, alignment: .center)
            }
            
            Button(action: {
                signIn()
            }) {
                Text("Sign In")
                    .font(.headline.bold())
                    .foregroundColor(.white)
                    .frame(maxWidth: .infinity)
                    .padding(16)
                    .background(Color.financeAccent)
                    .cornerRadius(12)
                    .shadow(color: Color.financeAccent.opacity(0.6), radius: 8, x: 0, y: 4)
            }
            .padding(.horizontal, 20)
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
            
            // Biometric authentication button - only shown on devices that support biometrics
            if biometricType != .none {
                Button(action: {
                    biometricSignIn()
                }) {
                    HStack {
                        Image(systemName: biometricType == .faceID ? "faceid" : "touchid")
                            .font(.title2)
                            .foregroundColor(Color.financeAccent)
                        Text("Sign in with \(biometricType == .faceID ? "Face ID" : "Touch ID")")
                            .fontWeight(.semibold)
                            .foregroundColor(Color.financeAccent)
                    }
                    .padding(16)
                    .frame(maxWidth: .infinity)
                    .background(
                        RoundedRectangle(cornerRadius: 12)
                            .stroke(Color.financeAccent, lineWidth: 2)
                    )
                }
                .padding(.horizontal, 20)
                .padding(.top, 24)
                .frame(maxWidth: 480)
                .frame(maxWidth: .infinity, alignment: .center)
                
                Text("You can sign in quickly using biometrics.")
                    .font(.footnote)
                    .foregroundColor(Color.financeSecondaryAccent)
                    .padding(.top, 24)
                    .frame(maxWidth: 480)
                    .frame(maxWidth: .infinity, alignment: .center)
            }
        }
        .frame(maxWidth: 480)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    // MARK: - Sign Up View
    private var signUpView: some View {
        VStack(spacing: 16) {
            Group {
                TextField("Full Name", text: $name)
                    .textContentType(.name)
                    .textFieldStyle(.roundedBorder)
                
                TextField("Email", text: $email)
                    .keyboardType(.emailAddress)
                    .autocapitalization(.none)
                    .textContentType(.emailAddress)
                    .textFieldStyle(.roundedBorder)
                
                SecureField("Password", text: $password)
                    .textContentType(.newPassword)
                    .textFieldStyle(.roundedBorder)
                
                SecureField("Confirm Password", text: $confirmPassword)
                    .textContentType(.newPassword)
                    .textFieldStyle(.roundedBorder)
                
                TextField("Age", text: $age)
                    .keyboardType(.numberPad)
                    .textFieldStyle(.roundedBorder)
                
                TextField("Gender", text: $gender)
                    .textFieldStyle(.roundedBorder)
                
                TextField("Marital Status", text: $maritalStatus)
                    .textFieldStyle(.roundedBorder)
                
                TextField("Phone Number", text: $number)
                    .keyboardType(.phonePad)
                    .textFieldStyle(.roundedBorder)
                
                TextField("Monthly Salary (₹)", text: $monthlySalaryInput)
                    .keyboardType(.decimalPad)
                    .textFieldStyle(.roundedBorder)
            }
            .padding(.horizontal, 20)
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
            
            if showError {
                Text(errorMessage.isEmpty ? "Please check your inputs and try again." : errorMessage)
                    .foregroundColor(.red)
                    .padding(.horizontal, 20)
                    .multilineTextAlignment(.center)
                    .frame(maxWidth: 480)
                    .frame(maxWidth: .infinity, alignment: .center)
            }
            
            Button(action: {
                createAccount()
            }) {
                Text("Create Account")
                    .font(.headline.bold())
                    .foregroundColor(.white)
                    .frame(maxWidth: .infinity)
                    .padding(16)
                    .background(Color.financeAccent)
                    .cornerRadius(12)
                    .shadow(color: Color.financeAccent.opacity(0.6), radius: 8, x: 0, y: 4)
            }
            .padding(.horizontal, 20)
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
        }
        .frame(maxWidth: 480)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    // MARK: - Social Login Buttons
    private var socialLoginButtons: some View {
        VStack(spacing: 16) {
            Text("Or continue with")
                .foregroundColor(Color.financeSecondaryAccent)
                .font(.subheadline)
                .frame(maxWidth: .infinity, alignment: .center)
            
            HStack(spacing: 20) {
                Button(action: {
                    socialAlertMessage = "Demo: Sign in with Apple is not implemented."
                    showingSocialAlert = true
                    // In production, implement Sign in with Apple here.
                }) {
                    HStack {
                        Image(systemName: "applelogo")
                            .font(.title2)
                            .foregroundColor(.white)
                        Text("Sign in with Apple")
                            .fontWeight(.semibold)
                            .foregroundColor(.white)
                    }
                    .padding(16)
                    .frame(maxWidth: .infinity)
                    .background(Color.black)
                    .cornerRadius(12)
                }
                
                Button(action: {
                    socialAlertMessage = "Demo: Sign in with Google is not implemented."
                    showingSocialAlert = true
                    // In production, implement Google Sign-In here.
                }) {
                    HStack {
                        Image("google_logo")
                            .resizable()
                            .frame(width: 20, height: 20)
                            .cornerRadius(3)
                        Text("Sign in with Google")
                            .fontWeight(.semibold)
                            .foregroundColor(Color.black)
                    }
                    .padding(16)
                    .frame(maxWidth: .infinity)
                    .background(Color.white)
                    .overlay(
                        RoundedRectangle(cornerRadius: 12)
                            .stroke(Color.gray.opacity(0.5), lineWidth: 1)
                    )
                    .shadow(color: Color.gray.opacity(0.2), radius: 3, x: 0, y: 2)
                }
            }
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
        }
        .frame(maxWidth: 480)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    // MARK: - Actions
    
    private func signIn() {
        hideError()
        guard !email.trimmingCharacters(in: .whitespaces).isEmpty,
              !password.isEmpty else {
            showErrorWith("Please enter both email and password.")
            return
        }
        
        if let error = appState.signIn(email: email, password: password) {
            showErrorWith(error)
        } else {
            dismiss()
        }
    }
    
    private func createAccount() {
        hideError()
        
        // Validate all mandatory fields
        guard !name.trimmingCharacters(in: .whitespaces).isEmpty,
              !email.trimmingCharacters(in: .whitespaces).isEmpty,
              !password.isEmpty,
              !confirmPassword.isEmpty,
              password == confirmPassword,
              !monthlySalaryInput.isEmpty,
              let monthlySalaryValue = Double(monthlySalaryInput) else {
            showErrorWith("Please fill all fields correctly and ensure passwords match.")
            return
        }
        
        // Optionally validate email format here
        
        if let error = appState.createUser(name: name,
                                           age: age,
                                           gender: gender,
                                           maritalStatus: maritalStatus,
                                           number: number,
                                           email: email,
                                           password: password,
                                           monthlySalary: monthlySalaryValue) {
            showErrorWith(error)
        } else {
            dismiss()
        }
    }
    
    private func showErrorWith(_ message: String) {
        errorMessage = message
        biometricError = nil
        showError = true
    }
    
    private func hideError() {
        errorMessage = ""
        biometricError = nil
        showError = false
    }
    
    // MARK: - Biometric Authentication Helpers
    
    private var biometricType: LABiometryType {
        let context = LAContext()
        var error: NSError?
        if context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
            return context.biometryType
        }
        return .none
    }
    
    /// Attempts to authenticate user via Face ID / Touch ID biometrics.
    /// On success, signs in the most recently used account (if available).
    private func biometricSignIn() {
        hideError()
        let context = LAContext()
        context.localizedCancelTitle = "Cancel"
        
        var authError: NSError?
        let reason = biometricType == .faceID ? "Sign in with Face ID" : "Sign in with Touch ID"
        
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &authError) else {
            biometricError = "Biometric authentication is not available."
            return
        }
        
        isAuthenticatingBiometric = true
        
        context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, localizedReason: reason) { success, evaluateError in
            DispatchQueue.main.async {
                isAuthenticatingBiometric = false
                
                if success {
                    // Sign in the most recently used (or first) user if available
                    if let lastUser = appState.users.last {
                        appState.currentUser = lastUser
                        appState.saveCurrentUserId()
                        dismiss()
                    } else {
                        biometricError = "No saved users found for biometric sign-in."
                    }
                } else {
                    let message = evaluateError?.localizedDescription ?? "Biometric authentication failed."
                    biometricError = message
                }
            }
        }
    }
}

// MARK: - Main Tab View

struct MainTabView: View {
    @State private var selectedTab = 0
    @EnvironmentObject var appState: AppState
    
    var body: some View {
        TabView(selection: $selectedTab) {
            DashboardView()
                .tabItem {
                    Label("Home", systemImage: "house.fill")
                }
                .tag(0)
            
            ReportsView()
                .tabItem {
                    Label("Reports", systemImage: "chart.bar.fill")
                }
                .tag(1)
            
            InsightsView()
                .tabItem {
                    Label("Insights", systemImage: "lightbulb.fill")
                }
                .tag(2)
            
            ProfileView()
                .tabItem {
                    Label("Profile", systemImage: "person.fill")
                }
                .tag(3)
        }
        .background(FinanceBackground())
        .ignoresSafeArea()
    }
}

// MARK: - 3. Dashboard / Home Screen

struct DashboardView: View {
    @EnvironmentObject private var appState: AppState
    @State private var isAddingExpense: Bool = false
    
    // MARK: - New State for Enhancements
    @State private var selectedCategoryFilter: String = "All" // For filtering cards and pie chart
    @State private var selectedFinancialCard: FinancialData? = nil // For showing detail sheet on card tap
    @Namespace private var cardNamespace // For matched geometry effect
    
    // Generate categories dynamically from currentUser expenses
    private var expenseCategories: [String] {
        let cats = Set(appState.currentUser?.expenses.map { $0.category } ?? [])
        return Array(cats).sorted()
    }
    
    // Filtered expenses based on selected category filter
    private var filteredExpenses: [Expense] {
        var filtered = appState.currentUser?.expenses ?? []
        
        // Filter by category
        if selectedCategoryFilter != "All" {
            filtered = filtered.filter { $0.category == selectedCategoryFilter }
        }
        
        return filtered
    }
    
    private func makeFinancialCards() -> [FinancialData] {
        let totalFilteredExpenses: Double = filteredExpenses.reduce(0) { sum, expense in
            sum + expense.amount
        }
        let totalIncome: Double = appState.totalIncome
        let numberOfExpenses: Double = Double(filteredExpenses.count)
        
        let expensesCard = FinancialData(title: "Total Expenses", value: totalFilteredExpenses, iconName: "arrow.down.circle.fill", color: .red, prefix: "₹")
        let incomeCard = FinancialData(title: "Total Income", value: totalIncome, iconName: "arrow.up.circle.fill", color: .green, prefix: "₹")
        let numberExpensesCard = FinancialData(title: "Number of Expenses", value: numberOfExpenses, iconName: "list.number", color: .blue, prefix: nil)
        
        return [expensesCard, incomeCard, numberExpensesCard]
    }
    
    // Trend data - static as before
    private let trendData: [FinancialTrend] = [
        FinancialTrend(month: "Jan", expenses: 22000, income: 38000),
        FinancialTrend(month: "Feb", expenses: 24500, income: 40000),
        FinancialTrend(month: "Mar", expenses: 21000, income: 39000),
        FinancialTrend(month: "Apr", expenses: 26000, income: 42000),
        FinancialTrend(month: "May", expenses: 23500, income: 41000)
    ]
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 24) {
                    
                    categoryFilterSection
                    
                    overviewHeaderSection
                    
                    financialCardsSection
                    
                    trendsHeaderSection
                    
                    expensesVsIncomeSection
                    
                    addExpenseButton
                }
                .padding(.top, 40)
                .padding(.bottom, 40)
                .frame(maxWidth: 640)
                .frame(maxWidth: .infinity, alignment: .center)
            }
            .background(FinanceBackground())
            .ignoresSafeArea()
            .navigationTitle("Dashboard")
            .navigationBarHidden(true)
            .fullScreenCover(isPresented: $isAddingExpense) {
                AddExpenseView()
            }
            // MARK: - Financial Card Detail Sheet
            .sheet(item: $selectedFinancialCard) { card in
                FinancialCardDetailView(data: card, namespace: cardNamespace)
            }
        }
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private var categoryFilterSection: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("Category Filter")
                .font(.headline)
                .foregroundColor(Color.financeText)
                .padding(.horizontal, 20)
                .multilineTextAlignment(.center)
                .frame(maxWidth: .infinity, alignment: .center)
            
            Picker("Select Category", selection: $selectedCategoryFilter) {
                Text("All").tag("All")
                ForEach(expenseCategories, id: \.self) { category in
                    Text(category).tag(category)
                }
            }
            .pickerStyle(SegmentedPickerStyle())
            .padding(.horizontal, 20)
            .tint(Color.financeSecondaryAccent)
        }
        .frame(maxWidth: 640)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private var overviewHeaderSection: some View {
        Text("Overview")
            .font(.title2.bold())
            .foregroundColor(Color.financeText)
            .padding(.horizontal, 20)
            .multilineTextAlignment(.center)
            .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private var financialCardsSection: some View {
        let cards = makeFinancialCards()
        return VStack(spacing: 16) {
            HStack(spacing: 16) {
                FinancialCardView(data: cards[0])
                    // Matched geometry effect for animation
                    .matchedGeometryEffect(id: cards[0].id, in: cardNamespace)
                    // Card tap gesture to show detail sheet
                    .onTapGesture {
                        withAnimation(.spring(response: 0.5, dampingFraction: 0.75)) {
                            selectedFinancialCard = cards[0]
                        }
                    }
                    // Card styling for tap/highlight and hover effect
                    .background(
                        RoundedRectangle(cornerRadius: 12)
                            .fill(Color.financeBackground)
                            .shadow(color: selectedFinancialCard?.id == cards[0].id ? cards[0].color.opacity(0.5) : Color.gray.opacity(0.1), radius: selectedFinancialCard?.id == cards[0].id ? 12 : 5, x: 0, y: 5)
                            .overlay(
                                RoundedRectangle(cornerRadius: 12)
                                    .stroke(selectedFinancialCard?.id == cards[0].id ? cards[0].color.opacity(0.8) : Color.clear, lineWidth: 2)
                                    .blur(radius: selectedFinancialCard?.id == cards[0].id ? 4 : 0)
                            )
                    )
                    .scaleEffect(selectedFinancialCard?.id == cards[0].id ? 1.05 : 1)
                    .animation(.easeInOut(duration: 0.3), value: selectedFinancialCard?.id == cards[0].id)
                
                FinancialCardView(data: cards[1])
                    .matchedGeometryEffect(id: cards[1].id, in: cardNamespace)
                    .onTapGesture {
                        withAnimation(.spring(response: 0.5, dampingFraction: 0.75)) {
                            selectedFinancialCard = cards[1]
                        }
                    }
                    .background(
                        RoundedRectangle(cornerRadius: 12)
                            .fill(Color.financeBackground)
                            .shadow(color: selectedFinancialCard?.id == cards[1].id ? cards[1].color.opacity(0.5) : Color.gray.opacity(0.1), radius: selectedFinancialCard?.id == cards[1].id ? 12 : 5, x: 0, y: 5)
                            .overlay(
                                RoundedRectangle(cornerRadius: 12)
                                    .stroke(selectedFinancialCard?.id == cards[1].id ? cards[1].color.opacity(0.8) : Color.clear, lineWidth: 2)
                                    .blur(radius: selectedFinancialCard?.id == cards[1].id ? 4 : 0)
                            )
                    )
                    .scaleEffect(selectedFinancialCard?.id == cards[1].id ? 1.05 : 1)
                    .animation(.easeInOut(duration: 0.3), value: selectedFinancialCard?.id == cards[1].id)
            }
            .padding(.horizontal, 20)
            
            HStack {
                Spacer()
                FinancialCardView(data: cards[2])
                    .matchedGeometryEffect(id: cards[2].id, in: cardNamespace)
                    .onTapGesture {
                        withAnimation(.spring(response: 0.5, dampingFraction: 0.75)) {
                            selectedFinancialCard = cards[2]
                        }
                    }
                    .background(
                        RoundedRectangle(cornerRadius: 12)
                            .fill(Color.financeBackground)
                            .shadow(color: selectedFinancialCard?.id == cards[2].id ? cards[2].color.opacity(0.5) : Color.gray.opacity(0.1), radius: selectedFinancialCard?.id == cards[2].id ? 12 : 5, x: 0, y: 5)
                            .overlay(
                                RoundedRectangle(cornerRadius: 12)
                                    .stroke(selectedFinancialCard?.id == cards[2].id ? cards[2].color.opacity(0.8) : Color.clear, lineWidth: 2)
                                    .blur(radius: selectedFinancialCard?.id == cards[2].id ? 4 : 0)
                            )
                    )
                    .scaleEffect(selectedFinancialCard?.id == cards[2].id ? 1.05 : 1)
                    .animation(.easeInOut(duration: 0.3), value: selectedFinancialCard?.id == cards[2].id)
                Spacer()
            }
            .padding(.horizontal, 20)
        }
        .frame(maxWidth: 640)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private var trendsHeaderSection: some View {
        Text("Trends")
            .font(.title2.bold())
            .foregroundColor(Color.financeText)
            .padding(.horizontal, 20)
            .padding(.top, 24)
            .multilineTextAlignment(.center)
            .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private var expensesVsIncomeSection: some View {
        VStack(alignment: .leading, spacing: 10) {
            Text("Expenses vs Income")
                .font(.headline)
                .foregroundColor(Color.financeText)
                .padding(.horizontal, 20)
                .frame(maxWidth: .infinity, alignment: .center)
            
            Chart {
                ForEach(trendData) { trend in
                    LineMark(
                        x: .value("Month", trend.month),
                        y: .value("Expenses", trend.expenses)
                    )
                    .foregroundStyle(Color.red)
                    .symbol(.circle)
                    
                    LineMark(
                        x: .value("Month", trend.month),
                        y: .value("Income", trend.income)
                    )
                    .foregroundStyle(Color.green)
                    .symbol(.square)
                }
            }
            .frame(height: 200)
            .chartLegend(.hidden)
            .padding(16)
            .background(Color.financeBackground)
            .cornerRadius(12)
            .shadow(color: .gray.opacity(0.1), radius: 5, x: 0, y: 5)
        }
        .frame(maxWidth: 640)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private var addExpenseButton: some View {
        Button(action: {
            isAddingExpense = true
        }) {
            HStack {
                Image(systemName: "plus.circle.fill")
                    .resizable()
                    .frame(width: 24, height: 24)
                Text("Add Expense / Scan Receipt")
                    .font(.headline)
            }
            .foregroundColor(.white)
            .padding(16)
            .frame(maxWidth: .infinity)
            .background(Color.financeAccent)
            .cornerRadius(12)
            .padding(.horizontal, 20)
            .padding(.bottom, 80)
        }
        .frame(maxWidth: 480)
        .frame(maxWidth: .infinity, alignment: .center)
    }
}

struct FinancialCardView: View {
    let data: FinancialData
    
    var body: some View {
        VStack(spacing: 12) {
            Image(systemName: data.iconName)
                .font(.largeTitle)
                .foregroundColor(data.color)
            Text(data.title)
                .font(.headline)
                .foregroundColor(Color.financeText)
            Text("\(data.prefix ?? "")\(String(format: "%.2f", data.value))")
                .font(.title2.bold())
                .foregroundColor(data.color)
        }
        .padding(16)
        .frame(maxWidth: 280)
    }
}

struct FinancialCardDetailView: View {
    let data: FinancialData
    var namespace: Namespace.ID
    @Environment(\.dismiss) private var dismiss
    
    var body: some View {
        VStack(spacing: 24) {
            // Large icon with matched geometry effect
            Image(systemName: data.iconName)
                .resizable()
                .scaledToFit()
                .frame(width: 80, height: 80)
                .foregroundColor(data.color)
                .matchedGeometryEffect(id: data.id, in: namespace)
            
            Text(data.title)
                .font(.title.bold())
                .foregroundColor(Color.financeText)
                .multilineTextAlignment(.center)
                .frame(maxWidth: .infinity, alignment: .center)
            
            Text("\(data.prefix ?? "")\(String(format: "%.2f", data.value))")
                .font(.largeTitle.bold())
                .foregroundColor(data.color)
            
            Divider()
            
            // Example detailed info - static or could be extended
            Text("Detailed insights and breakdowns for \(data.title.lowercased()) will appear here. This can include trends, tips, and actionable advice.")
                .font(.body)
                .foregroundColor(Color.financeText)
                .multilineTextAlignment(.center)
                .padding(16)
                .frame(maxWidth: 480)
            
            Spacer()
            
            Button("Close") {
                withAnimation(.spring(response: 0.5, dampingFraction: 0.75)) {
                    dismiss()
                }
            }
            .buttonStyle(.borderedProminent)
            .tint(Color.financeAccent)
            .controlSize(.large)
            .padding(.bottom, 24)
        }
        .padding(16)
        .background(Color.financeBackground)
        .ignoresSafeArea()
        .frame(maxWidth: 480)
        .frame(maxWidth: .infinity, alignment: .center)
    }
}

// MARK: - 4. Add Expense Screen

struct AddExpenseView: View {
    @State private var amount: String = ""
    @State private var category: String = "Food"
    @State private var notes: String = ""
    @State private var expenseDate: Date = Date()
    @EnvironmentObject var appState: AppState
    
    let categories = ["Food", "Rent", "Shopping", "Transport", "Bills"]
    
    @Environment(\.dismiss) var dismiss
    
    var body: some View {
        NavigationStack {
            Form {
                Section(header: Text("Expense Details")) {
                    TextField("Amount (₹)", text: $amount)
                        .keyboardType(.decimalPad)
                    
                    Picker("Category", selection: $category) {
                        ForEach(categories, id: \.self) {
                            Text($0)
                        }
                    }
                    .tint(Color.financeSecondaryAccent)
                    
                    DatePicker("Date", selection: $expenseDate, displayedComponents: .date)
                    
                    TextField("Notes", text: $notes, axis: .vertical)
                        .lineLimit(3)
                }
                
                Section {
                    Button("Scan Receipt") {
                        // VisionKit implementation would go here.
                    }
                    .tint(Color.financeSecondaryAccent)
                    
                    Button("Save Expense") {
                        if let amountValue = Double(amount) {
                            let newExpense = Expense(category: category, amount: amountValue, date: expenseDate)
                            appState.addExpense(newExpense)
                        }
                        dismiss()
                    }
                    .tint(Color.financeAccent)
                }
            }
            .padding(.top, 32)
            .padding(.bottom, 32)
            .background(FinanceBackground())
            .ignoresSafeArea()
            .navigationTitle("Add Expense")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") {
                        dismiss()
                    }
                    .tint(Color.financeSecondaryAccent)
                }
            }
            .frame(maxWidth: 480)
            .frame(maxWidth: .infinity, alignment: .center)
        }
        .padding(16)
        .frame(maxWidth: .infinity, alignment: .center)
    }
}

// MARK: - 5. Reports & Tabulation Screen

struct ReportsView: View {
    @State private var selectedTab = "Monthly"
    @EnvironmentObject var appState: AppState
    
    @State private var selectedSlice: PieSlice?
    
    // Helper: Compute pie slices for current filtered expenses (all expenses here)
    private func makePieSlices() -> [PieSlice] {
        let expenses = appState.currentUser?.expenses ?? []
        let categories = Dictionary(grouping: expenses, by: { $0.category })
        let colors: [Color] = [.blue, .red, .green, .orange, .purple, .mint, .pink, .yellow, .cyan]
        var colorIdx = 0
        return categories.map { (cat, exps) in
            defer { colorIdx = (colorIdx + 1) % colors.count }
            return PieSlice(category: cat, amount: exps.reduce(0) { $0 + $1.amount }, color: colors[colorIdx])
        }
    }
    
    let monthlyTrends = [
        FinancialTrend(month: "Jan", expenses: 22000, income: 38000),
        FinancialTrend(month: "Feb", expenses: 24500, income: 40000),
        FinancialTrend(month: "Mar", expenses: 21000, income: 39000),
        FinancialTrend(month: "Apr", expenses: 26000, income: 42000),
        FinancialTrend(month: "May", expenses: 23500, income: 41000)
    ]
    
    private static let expenseDateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateFormat = "dd MMM yyyy"
        return formatter
    }()
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    
                    Text("Reports")
                        .font(.largeTitle.bold())
                        .multilineTextAlignment(.center)
                        .frame(maxWidth: .infinity, alignment: .center)
                        .padding(.bottom, 8)
                    
                    Picker("Time Period", selection: $selectedTab) {
                        Text("Daily").tag("Daily")
                        Text("Weekly").tag("Weekly")
                        Text("Monthly").tag("Monthly")
                    }
                    .pickerStyle(SegmentedPickerStyle())
                    .padding(.horizontal, 20)
                    .tint(Color.financeSecondaryAccent)
                    .frame(maxWidth: 640)
                    .frame(maxWidth: .infinity, alignment: .center)
                    
                    VStack(alignment: .leading) {
                        Text("Category Split")
                            .font(.headline)
                            .foregroundColor(Color.financeText)
                            .frame(maxWidth: .infinity, alignment: .center)
                        
                        Chart {
                            ForEach(appState.currentUser?.expenses ?? [], id: \.id) { expense in
                                SectorMark(
                                    angle: .value("Amount", expense.amount),
                                    innerRadius: .ratio(0.6)
                                )
                                .foregroundStyle(by: .value("Category", expense.category))
                            }
                        }
                        .frame(height: 250)
                        .padding(16)
                        .background(Color.financeBackground)
                        .cornerRadius(12)
                        .shadow(color: .gray.opacity(0.1), radius: 5, x: 0, y: 5)
                    }
                    .padding(.horizontal, 20)
                    .frame(maxWidth: 640)
                    .frame(maxWidth: .infinity, alignment: .center)
                    
                    // New section for All Expenses list
                    VStack(alignment: .leading, spacing: 10) {
                        Text("All Expenses")
                            .font(.headline)
                            .foregroundColor(Color.financeText)
                            .padding(.bottom, 24)
                            .frame(maxWidth: .infinity, alignment: .center)
                        
                        if let expenses = appState.currentUser?.expenses, !expenses.isEmpty {
                            ForEach(expenses.sorted(by: { $0.date > $1.date })) { expense in
                                HStack {
                                    Text("₹\(String(format: "%.2f", expense.amount))")
                                        .fontWeight(.semibold)
                                        .frame(minWidth: 80, alignment: .leading)
                                    Text(expense.category)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    Spacer()
                                    Text(Self.expenseDateFormatter.string(from: expense.date))
                                        .foregroundColor(Color.financeSecondaryAccent)
                                        .font(.caption)
                                }
                                .padding(.vertical, 16)
                                Divider()
                            }
                        } else {
                            Text("No expenses recorded yet.")
                                .foregroundColor(Color.financeSecondaryAccent)
                                .italic()
                        }
                    }
                    .padding(16)
                    .background(Color.financeBackground)
                    .cornerRadius(12)
                    .shadow(color: .gray.opacity(0.1), radius: 5, x: 0, y: 5)
                    .frame(maxWidth: 640)
                    .frame(maxWidth: .infinity, alignment: .center)
                }
                .padding(.top, 40)
                .padding(.bottom, 40)
                .padding(20)
                .frame(maxWidth: 640)
                .frame(maxWidth: .infinity, alignment: .center)
            }
            .background(FinanceBackground())
            .ignoresSafeArea()
            .navigationTitle("Reports")
        }
        .frame(maxWidth: .infinity, alignment: .center)
    }
}

// MARK: - 6. Insights Screen

struct InsightsView: View {
    @EnvironmentObject var appState: AppState
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 20) {
                    
                    Text("Your Insights")
                        .font(.largeTitle.bold())
                        .multilineTextAlignment(.center)
                        .frame(maxWidth: .infinity, alignment: .center)
                        .padding(.bottom, 8)
                    
                    // Spending Summary Insight
                    VStack(alignment: .leading, spacing: 8) {
                        Text("Spending Habits")
                            .font(.title2.bold())
                            .foregroundColor(Color.financeText)
                            .frame(maxWidth: .infinity, alignment: .center)
                        
                        let total = appState.totalExpenses
                        let salary = appState.totalIncome
                        let percentage = salary == 0 ? 0 : (total / salary * 100)
                        
                        InsightCard(
                            title: "Monthly Spending",
                            value: "₹\(String(format: "%.2f", total))",
                            icon: "chart.pie.fill",
                            color: .purple,
                            details: "You've spent **\(Int(percentage))%** of your monthly income so far. Keep an eye on your budget to stay on track."
                        )
                    }
                    .padding(.horizontal, 20)
                    
                    // Top Categories Insight
                    VStack(alignment: .leading, spacing: 8) {
                        Text("Top Categories")
                            .font(.title2.bold())
                            .foregroundColor(Color.financeText)
                            .frame(maxWidth: .infinity, alignment: .center)
                        
                        if let topCategory = appState.currentUser?.expenses.sorted(by: { $0.amount > $1.amount }).first {
                            InsightCard(
                                title: "Biggest Expense",
                                value: topCategory.category,
                                icon: "bag.fill",
                                color: .orange,
                                details: "Your largest expense category is **\(topCategory.category)**. Consider setting a budget for this area."
                            )
                        } else {
                            Text("No expenses recorded yet.")
                                .foregroundColor(Color.financeSecondaryAccent)
                        }
                    }
                    .padding(.horizontal, 20)
                    
                    // Savings Potential Insight
                    VStack(alignment: .leading, spacing: 8) {
                        Text("Savings Potential")
                            .font(.title2.bold())
                            .foregroundColor(Color.financeText)
                            .frame(maxWidth: .infinity, alignment: .center)
                        
                        let savings = appState.totalIncome - appState.totalExpenses
                        let color: Color = savings >= 0 ? .green : .red
                        let prefix: String = savings >= 0 ? "You're saving" : "You're overspending"
                        
                        InsightCard(
                            title: "Current Savings",
                            value: "₹\(String(format: "%.2f", abs(savings)))",
                            icon: "dollarsign.circle.fill",
                            color: color,
                            details: "\(prefix). Set a savings goal to track your progress."
                        )
                    }
                    .padding(.horizontal, 20)
                }
                .padding(.top, 40)
                .padding(.bottom, 40)
                .padding(.vertical, 24)
                .frame(maxWidth: 640)
                .frame(maxWidth: .infinity, alignment: .center)
            }
            .background(FinanceBackground())
            .ignoresSafeArea()
            .navigationTitle("Insights")
        }
        .frame(maxWidth: .infinity, alignment: .center)
    }
}

struct InsightCard: View {
    let title: String
    let value: String
    let icon: String
    let color: Color
    let details: LocalizedStringKey
    
    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            HStack(spacing: 12) {
                Image(systemName: icon)
                    .font(.title2)
                    .foregroundColor(color)
                    .frame(width: 40, height: 40)
                    .background(color.opacity(0.2))
                    .cornerRadius(8)
                
                VStack(alignment: .leading) {
                    Text(title)
                        .font(.subheadline)
                        .foregroundColor(Color.financeSecondaryAccent)
                    Text(value)
                        .font(.title.bold())
                        .foregroundColor(Color.financeText)
                }
            }
            Text(details)
                .font(.callout)
                .foregroundColor(Color.financeText)
        }
        .padding(16)
        .background(Color.financeBackground)
        .cornerRadius(12)
        .shadow(color: .gray.opacity(0.1), radius: 5, x: 0, y: 5)
        .frame(maxWidth: 480)
    }
}

// MARK: - 7. Profile Screen

struct ProfileView: View {
    @EnvironmentObject var appState: AppState
    @Environment(\.dismiss) var dismiss
    
    @State private var isEditing: Bool = false
    
    @State private var name: String = ""
    @State private var age: String = ""
    @State private var gender: String = ""
    @State private var maritalStatus: String = ""
    @State private var number: String = ""
    @State private var monthlySalaryInput: String = ""
    
    @State private var errorMessage: String = ""
    @State private var showErrorMessage: Bool = false
    
    var body: some View {
        NavigationStack {
            Group {
                if let user = appState.currentUser {
                    if isEditing {
                        Form {
                            Section(header: Text("Personal Details")) {
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Name")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Full Name", text: $name)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Email")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Email", text: .constant(user.email))
                                        .disabled(true)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Monthly Salary (₹)")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Monthly Salary (₹)", text: $monthlySalaryInput)
                                        .keyboardType(.decimalPad)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Age")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Age", text: $age)
                                        .keyboardType(.numberPad)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Gender")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Gender", text: $gender)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Marital Status")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Marital Status", text: $maritalStatus)
                                }
                                VStack(alignment: .leading, spacing: 4) {
                                    Text("Phone Number")
                                        .font(.caption)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                    TextField("Phone Number", text: $number)
                                        .keyboardType(.phonePad)
                                }
                            }
                            
                            if showErrorMessage {
                                Text(errorMessage)
                                    .foregroundColor(.red)
                            }
                            
                            Section {
                                Button("Save") {
                                    saveChanges(for: user)
                                }
                                .tint(Color.financeAccent)
                                Button("Cancel") {
                                    isEditing = false
                                    showErrorMessage = false
                                }
                                .foregroundColor(.red)
                            }
                        }
                        .padding(.top, 40)
                        .padding(.bottom, 40)
                        .padding(16)
                        .frame(maxWidth: 480)
                        .frame(maxWidth: .infinity, alignment: .center)
                    } else {
                        List {
                            Section(header: Text("Profile Information")) {
                                HStack {
                                    Text("Name")
                                    Spacer()
                                    Text(user.name)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                HStack {
                                    Text("Email")
                                    Spacer()
                                    Text(user.email)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                HStack {
                                    Text("Monthly Salary")
                                    Spacer()
                                    Text("₹\(String(format: "%.2f", user.monthlySalary))")
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                HStack {
                                    Text("Age")
                                    Spacer()
                                    Text(user.age)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                HStack {
                                    Text("Gender")
                                    Spacer()
                                    Text(user.gender)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                HStack {
                                    Text("Marital Status")
                                    Spacer()
                                    Text(user.maritalStatus)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                                HStack {
                                    Text("Phone Number")
                                    Spacer()
                                    Text(user.number)
                                        .foregroundColor(Color.financeSecondaryAccent)
                                }
                            }
                            
                            Section {
                                Button("Sign Out") {
                                    appState.signOut()
                                }
                                .foregroundColor(.red)
                            }
                        }
                        .padding(16)
                        .frame(maxWidth: 480)
                        .frame(maxWidth: .infinity, alignment: .center)
                    }
                } else {
                    Text("User not signed in.")
                        .foregroundColor(Color.financeSecondaryAccent)
                        .frame(maxWidth: .infinity, alignment: .center)
                }
            }
            .background(FinanceBackground())
            .ignoresSafeArea()
            .navigationTitle("Profile")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                if appState.currentUser != nil && !isEditing {
                    ToolbarItem(placement: .navigationBarTrailing) {
                        Button("Edit") {
                            startEditing()
                        }
                        .tint(Color.financeAccent)
                    }
                }
            }
        }
        .padding(.top, 40)
        .padding(.bottom, 40)
        .frame(maxWidth: .infinity, alignment: .center)
    }
    
    private func startEditing() {
        guard let user = appState.currentUser else { return }
        name = user.name
        age = user.age
        gender = user.gender
        maritalStatus = user.maritalStatus
        number = user.number
        monthlySalaryInput = String(format: "%.2f", user.monthlySalary)
        isEditing = true
        showErrorMessage = false
        errorMessage = ""
    }
    
    private func saveChanges(for user: User) {
        // Validate inputs
        guard !name.trimmingCharacters(in: .whitespaces).isEmpty else {
            errorMessage = "Name cannot be empty."
            showErrorMessage = true
            return
        }
        guard let monthlySalaryValue = Double(monthlySalaryInput), monthlySalaryValue >= 0 else {
            errorMessage = "Enter a valid monthly salary."
            showErrorMessage = true
            return
        }
        
        // Age can be empty but if present validate numeric
        if !age.trimmingCharacters(in: .whitespaces).isEmpty {
            if Int(age) == nil {
                errorMessage = "Age must be a number."
                showErrorMessage = true
                return
            }
        }
        
        // All validations passed, proceed to update
        var updatedUser = user
        updatedUser.name = name.trimmingCharacters(in: .whitespaces)
        updatedUser.age = age.trimmingCharacters(in: .whitespaces)
        updatedUser.gender = gender.trimmingCharacters(in: .whitespaces)
        updatedUser.maritalStatus = maritalStatus.trimmingCharacters(in: .whitespaces)
        updatedUser.number = number.trimmingCharacters(in: .whitespaces)
        updatedUser.monthlySalary = monthlySalaryValue
        
        appState.updateCurrentUser(updatedUser)
        isEditing = false
        showErrorMessage = false
        errorMessage = ""
    }
}

// MARK: - Color Extensions

struct FinanceBackground: View {
    @State private var animate = false
    var body: some View {
        ZStack {
            // Dynamic angular gradient
            AngularGradient(
                gradient: Gradient(colors: [
                    Color.financeBackground,
                    Color.white,
                    Color.financeSecondaryAccent.opacity(0.18),
                    Color.financeAccent.opacity(0.22),
                    Color.financeBackground
                ]),
                center: .center
            )
            .blur(radius: 8)
            .ignoresSafeArea()

            // Glassy, animated blurred circles
            Circle()
                .fill(Color.financeSecondaryAccent.opacity(0.16))
                .frame(width: 340, height: 340)
                .blur(radius: 55)
                .offset(x: -120, y: animate ? -240 : -200)
                .animation(Animation.easeInOut(duration: 8).repeatForever(autoreverses: true), value: animate)
            Circle()
                .fill(Color.financeAccent.opacity(0.22))
                .frame(width: 240, height: 240)
                .blur(radius: 60)
                .offset(x: 160, y: animate ? 180 : 140)
                .animation(Animation.easeInOut(duration: 9).repeatForever(autoreverses: true), value: animate)
            Circle()
                .fill(Color.financeAccent.opacity(0.11))
                .frame(width: 180, height: 180)
                .blur(radius: 30)
                .offset(x: 100, y: -140)

            // Soft grain overlay
            Rectangle()
                .fill(Color.white.opacity(0.02))
                .background(
                    Image(systemName: "circle.fill")
                        .resizable()
                        .scaledToFill()
                        .frame(width: 600, height: 600)
                        .opacity(0.017)
                        .blur(radius: 2)
                        .offset(x: -30, y: 80)
                )
                .ignoresSafeArea()
        }
        .onAppear { animate = true }
    }
}

extension Color {
    static let financeBackground = Color(red: 0.96, green: 0.97, blue: 0.98) // #F5F7FA
    static let financeAccent = Color(red: 0.18, green: 0.42, blue: 0.31) // #2D6A4F
    static let financeSecondaryAccent = Color(red: 0.25, green: 0.57, blue: 0.42) // #40916C
    static let financeText = Color(red: 0.11, green: 0.26, blue: 0.20) // #1B4332
}

