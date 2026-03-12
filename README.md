```
package com.liam.walletai

import android.app.DatePickerDialog
import android.content.Intent
import android.graphics.BitmapFactory
import android.net.Uri
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.widget.ArrayAdapter
import android.widget.Button
import android.widget.EditText
import android.widget.ImageView
import android.widget.LinearLayout
import android.widget.Spinner
import android.widget.TextView
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AlertDialog
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.FileProvider
import androidx.core.widget.addTextChangedListener
import androidx.lifecycle.lifecycleScope
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.google.mlkit.vision.common.InputImage
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import com.liam.walletai.data.AccountDao
import com.liam.walletai.data.AccountEntity
import com.liam.walletai.data.AccountTotal
import com.liam.walletai.data.BudgetDao
import com.liam.walletai.data.BudgetEntity
import com.liam.walletai.data.CategoryDao
import com.liam.walletai.data.CategoryEntity
import com.liam.walletai.data.CategoryTotal
import com.liam.walletai.data.DatabaseProvider
import com.liam.walletai.data.RecurringExpenseDao
import com.liam.walletai.data.RecurringExpenseEntity
import com.liam.walletai.data.TransactionDao
import com.liam.walletai.data.TransactionEntity
import com.liam.walletai.ui.TransactionAdapter
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import org.json.JSONArray
import org.json.JSONObject
import java.io.File
import java.text.NumberFormat
import java.text.SimpleDateFormat
import java.util.Calendar
import java.util.Date
import java.util.Locale
import kotlin.math.min

class MainActivity : AppCompatActivity() {

    private enum class FilterType { ALL, WEEK, MONTH }

    private lateinit var tvWeekTotal: TextView
    private lateinit var tvMonthTotal: TextView
    private lateinit var tvEmptyState: TextView
    private lateinit var tvInsight: TextView
    private lateinit var tvBudgetStatus: TextView
    private lateinit var tvThreeMonthStats: TextView
    private lateinit var etSearch: EditText
    private lateinit var layoutCategorySummary: LinearLayout
    private lateinit var layoutBudgetSummary: LinearLayout
    private lateinit var layoutAccountSummary: LinearLayout
    private lateinit var recyclerTransactions: RecyclerView
    private lateinit var transactionAdapter: TransactionAdapter

    private lateinit var btnFilterAll: Button
    private lateinit var btnFilterWeek: Button
    private lateinit var btnFilterMonth: Button
    private lateinit var btnSetBudget: Button
    private lateinit var btnExportCsv: Button
    private lateinit var btnGallery: Button
    private lateinit var btnRecurring: Button
    private lateinit var btnSettings: Button

    private lateinit var prefsManager: PrefsManager

    private val db by lazy { DatabaseProvider.get(this) }
    private val transactionDao: TransactionDao by lazy { db.transactionDao() }
    private val categoryDao: CategoryDao by lazy { db.categoryDao() }
    private val accountDao: AccountDao by lazy { db.accountDao() }
    private val budgetDao: BudgetDao by lazy { db.budgetDao() }
    private val recurringDao: RecurringExpenseDao by lazy { db.recurringExpenseDao() }

    private val categoryNames = mutableListOf<String>()
    private val accountNames = mutableListOf<String>()

    private var currentFilter = FilterType.ALL
    private var currentSearchQuery = ""
    private var currentVisibleItems: List<TransactionEntity> = emptyList()

    private val scanLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == RESULT_OK) {
            val data = result.data
            val merchant = data?.getStringExtra("merchant").orEmpty()
            val rawText = data?.getStringExtra("rawText")
            val suggestedCategory = suggestCategoryFromOcr(merchant, rawText)

            showTransactionDialog(
                existing = null,
                merchant = merchant,
                amount = data?.getLongExtra("amount", 0L) ?: 0L,
                category = suggestedCategory,
                accountName = "Cash",
                note = "",
                receiptPath = data?.getStringExtra("receiptPath"),
                rawText = rawText,
                sourceType = "scan",
                transactionDate = System.currentTimeMillis()
            )
        }
    }

    private val galleryLauncher = registerForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri ->
        if (uri != null) handleGalleryImage(uri)
    }

    private val createBackupLauncher = registerForActivityResult(
        ActivityResultContracts.CreateDocument("application/json")
    ) { uri ->
        if (uri != null) {
            lifecycleScope.launch {
                try {
                    backupToUri(uri)
                    Toast.makeText(
                        this@MainActivity,
                        getString(R.string.toast_backup_success),
                        Toast.LENGTH_SHORT
                    ).show()
                } catch (e: Exception) {
                    Toast.makeText(
                        this@MainActivity,
                        "Backup gagal: ${e.message}",
                        Toast.LENGTH_LONG
                    ).show()
                }
            }
        }
    }

    private val restoreBackupLauncher = registerForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri ->
        if (uri != null) {
            lifecycleScope.launch {
                try {
                    restoreFromUri(uri)
                    Toast.makeText(
                        this@MainActivity,
                        getString(R.string.toast_restore_success),
                        Toast.LENGTH_SHORT
                    ).show()
                } catch (e: Exception) {
                    Toast.makeText(
                        this@MainActivity,
                        "Restore gagal: ${e.message}",
                        Toast.LENGTH_LONG
                    ).show()
                }
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        prefsManager = PrefsManager(this)
        if (!prefsManager.isOnboardingDone()) {
            startActivity(Intent(this, OnboardingActivity::class.java))
            finish()
            return
        }

        setContentView(R.layout.activity_main)

        tvWeekTotal = findViewById(R.id.tvWeekTotal)
        tvMonthTotal = findViewById(R.id.tvMonthTotal)
        tvEmptyState = findViewById(R.id.tvEmptyState)
        tvInsight = findViewById(R.id.tvInsight)
        tvBudgetStatus = findViewById(R.id.tvBudgetStatus)
        tvThreeMonthStats = findViewById(R.id.tvThreeMonthStats)
        etSearch = findViewById(R.id.etSearch)
        layoutCategorySummary = findViewById(R.id.layoutCategorySummary)
        layoutBudgetSummary = findViewById(R.id.layoutBudgetSummary)
        layoutAccountSummary = findViewById(R.id.layoutAccountSummary)
        recyclerTransactions = findViewById(R.id.recyclerTransactions)

        btnFilterAll = findViewById(R.id.btnFilterAll)
        btnFilterWeek = findViewById(R.id.btnFilterWeek)
        btnFilterMonth = findViewById(R.id.btnFilterMonth)
        btnSetBudget = findViewById(R.id.btnSetBudget)
        btnExportCsv = findViewById(R.id.btnExportCsv)
        btnGallery = findViewById(R.id.btnGallery)
        btnRecurring = findViewById(R.id.btnRecurring)
        btnSettings = findViewById(R.id.btnSettings)

        transactionAdapter = TransactionAdapter(
            onClick = { item -> showDetailDialog(item) },
            onLongClick = { item -> showDeleteDialog(item) }
        )

        recyclerTransactions.layoutManager = LinearLayoutManager(this)
        recyclerTransactions.adapter = transactionAdapter

        findViewById<Button>(R.id.btnAddManual).setOnClickListener {
            showTransactionDialog()
        }

        findViewById<Button>(R.id.btnScan).setOnClickListener {
            scanLauncher.launch(Intent(this, ScanReceiptActivity::class.java))
        }

        btnGallery.setOnClickListener {
            galleryLauncher.launch("image/*")
        }

        btnRecurring.setOnClickListener {
            showRecurringDialog()
        }

        btnSetBudget.setOnClickListener {
            showBudgetDialog()
        }

        btnExportCsv.setOnClickListener {
            exportCurrentCsv()
        }

        btnSettings.setOnClickListener {
            showSettingsDialog()
        }

        btnFilterAll.setOnClickListener {
            currentFilter = FilterType.ALL
            updateFilterButtons()
            lifecycleScope.launch { refreshAll() }
        }

        btnFilterWeek.setOnClickListener {
            currentFilter = FilterType.WEEK
            updateFilterButtons()
            lifecycleScope.launch { refreshAll() }
        }

        btnFilterMonth.setOnClickListener {
            currentFilter = FilterType.MONTH
            updateFilterButtons()
            lifecycleScope.launch { refreshAll() }
        }

        etSearch.addTextChangedListener { text ->
            currentSearchQuery = text?.toString().orEmpty()
            lifecycleScope.launch { refreshAll() }
        }

        lifecycleScope.launch {
            seedDefaults()
            loadMasters()
            updateFilterButtons()
            refreshAll()
        }
    }

    override fun onResume() {
        super.onResume()
        if (::prefsManager.isInitialized && prefsManager.isOnboardingDone()) {
            lifecycleScope.launch {
                loadMasters()
                refreshAll()
            }
        }
    }

    private fun updateFilterButtons() {
        btnFilterAll.alpha = if (currentFilter == FilterType.ALL) 1f else 0.6f
        btnFilterWeek.alpha = if (currentFilter == FilterType.WEEK) 1f else 0.6f
        btnFilterMonth.alpha = if (currentFilter == FilterType.MONTH) 1f else 0.6f
    }

    private suspend fun seedDefaults() {
        categoryDao.insertAll(
            listOf(
                CategoryEntity(name = "Makan"),
                CategoryEntity(name = "Transport"),
                CategoryEntity(name = "Belanja"),
                CategoryEntity(name = "Tagihan"),
                CategoryEntity(name = "Hiburan"),
                CategoryEntity(name = "Kesehatan"),
                CategoryEntity(name = "Lainnya")
            )
        )

        accountDao.insertAll(
            listOf(
                AccountEntity(name = "Cash"),
                AccountEntity(name = "BCA"),
                AccountEntity(name = "BRI"),
                AccountEntity(name = "Mandiri"),
                AccountEntity(name = "Dana"),
                AccountEntity(name = "OVO"),
                AccountEntity(name = "GoPay")
            )
        )
    }

    private suspend fun loadMasters() {
        categoryNames.clear()
        categoryNames.addAll(categoryDao.getAll().map { it.name })

        accountNames.clear()
        accountNames.addAll(accountDao.getAll().map { it.name })

        if (categoryNames.isEmpty()) categoryNames.add("Lainnya")
        if (accountNames.isEmpty()) accountNames.add("Cash")
    }

    private suspend fun refreshAll() {
        applyRecurringTransactionsForCurrentMonth()

        val now = System.currentTimeMillis()
        val weekStart = getStartOfWeek(now)
        val monthStart = getStartOfMonth(now)
        val monthEnd = getStartOfNextMonth(now)

        val weekTotal = transactionDao.getTotalFrom(weekStart)
        val monthTotal = transactionDao.getTotalFrom(monthStart)

        val query = currentSearchQuery.trim()
        val items = when (currentFilter) {
            FilterType.ALL -> {
                if (query.isBlank()) transactionDao.getAllTransactions()
                else transactionDao.searchAllTransactions(query)
            }
            FilterType.WEEK -> {
                if (query.isBlank()) transactionDao.getTransactionsBetween(weekStart, now + 1)
                else transactionDao.searchTransactionsBetween(weekStart, now + 1, query)
            }
            FilterType.MONTH -> {
                if (query.isBlank()) transactionDao.getTransactionsBetween(monthStart, now + 1)
                else transactionDao.searchTransactionsBetween(monthStart, now + 1, query)
            }
        }

        val monthCategoryTotals = transactionDao.getCategoryTotalsBetween(monthStart, monthEnd)
        val monthAccountTotals = transactionDao.getAccountTotalsBetween(monthStart, monthEnd)
        val monthTransactionCount = transactionDao.getTransactionCountBetween(monthStart, monthEnd)
        val budgets = budgetDao.getAll()

        currentVisibleItems = items

        tvWeekTotal.text = formatRupiah(weekTotal)
        tvMonthTotal.text = formatRupiah(monthTotal)
        transactionAdapter.submitList(items)

        val isEmpty = items.isEmpty()
        tvEmptyState.visibility = if (isEmpty) View.VISIBLE else View.GONE
        recyclerTransactions.visibility = if (isEmpty) View.GONE else View.VISIBLE

        renderCategorySummary(monthCategoryTotals, monthTotal)
        renderBudgetSummary(budgets, monthCategoryTotals)
        renderAccountSummary(monthAccountTotals, monthTotal)
        tvInsight.text = buildInsight(monthCategoryTotals, monthTotal, monthTransactionCount)
        tvBudgetStatus.text = buildBudgetInsight(budgets, monthCategoryTotals)
        tvThreeMonthStats.text = buildThreeMonthStats(now)
    }

    private suspend fun applyRecurringTransactionsForCurrentMonth() {
        val now = System.currentTimeMillis()
        val monthStart = getStartOfMonth(now)
        val monthEnd = getStartOfNextMonth(now)
        val recurringItems = recurringDao.getAllActive()

        recurringItems.forEach { recurring ->
            val count = transactionDao.countRecurringTransactionInPeriod(
                recurringId = recurring.id,
                startTime = monthStart,
                endTime = monthEnd
            )

            if (count == 0) {
                val targetDate = buildRecurringDateForCurrentMonth(recurring.dayOfMonth, now)

                transactionDao.insert(
                    TransactionEntity(
                        merchant = recurring.merchant,
                        category = recurring.category,
                        accountName = recurring.accountName,
                        amount = recurring.amount,
                        transactionDate = targetDate,
                        note = recurring.note,
                        receiptPath = null,
                        rawText = null,
                        sourceType = "recurring",
                        recurringId = recurring.id
                    )
                )
            }
        }
    }

    private fun buildRecurringDateForCurrentMonth(dayOfMonth: Int, now: Long): Long {
        val cal = Calendar.getInstance().apply { timeInMillis = now }
        cal.set(Calendar.DAY_OF_MONTH, 1)
        val maxDay = cal.getActualMaximum(Calendar.DAY_OF_MONTH)
        cal.set(Calendar.DAY_OF_MONTH, min(dayOfMonth, maxDay))
        cal.set(Calendar.HOUR_OF_DAY, 9)
        cal.set(Calendar.MINUTE, 0)
        cal.set(Calendar.SECOND, 0)
        cal.set(Calendar.MILLISECOND, 0)
        return cal.timeInMillis
    }

    private fun renderCategorySummary(items: List<CategoryTotal>, monthTotal: Long) {
        layoutCategorySummary.removeAllViews()

        if (items.isEmpty()) {
            val tv = TextView(this).apply {
                text = "Belum ada data kategori bulan ini"
                textSize = 14f
            }
            layoutCategorySummary.addView(tv)
            return
        }

        items.take(5).forEach { item ->
            val percent = if (monthTotal > 0) (item.total * 100 / monthTotal) else 0
            val tv = TextView(this).apply {
                text = "${item.category} • ${formatRupiah(item.total)} • $percent%"
                textSize = 14f
                setPadding(0, 0, 0, 14)
            }
            layoutCategorySummary.addView(tv)
        }
    }

    private fun renderBudgetSummary(
        budgets: List<BudgetEntity>,
        categoryTotals: List<CategoryTotal>
    ) {
        layoutBudgetSummary.removeAllViews()

        if (budgets.isEmpty()) {
            val tv = TextView(this).apply {
                text = "Belum ada budget kategori"
                textSize = 14f
            }
            layoutBudgetSummary.addView(tv)
            return
        }

        budgets.forEach { budget ->
            val spent = categoryTotals.firstOrNull { it.category == budget.category }?.total ?: 0L
            val remaining = budget.amount - spent
            val status = if (remaining >= 0) {
                "Sisa ${formatRupiah(remaining)}"
            } else {
                "Lebih ${formatRupiah(-remaining)}"
            }

            val tv = TextView(this).apply {
                text = "${budget.category} • Budget ${formatRupiah(budget.amount)} • Keluar ${formatRupiah(spent)} • $status"
                textSize = 14f
                setPadding(0, 0, 0, 14)
            }
            layoutBudgetSummary.addView(tv)
        }
    }

    private fun renderAccountSummary(
        accounts: List<AccountTotal>,
        monthTotal: Long
    ) {
        layoutAccountSummary.removeAllViews()

        if (accounts.isEmpty()) {
            val tv = TextView(this).apply {
                text = "Belum ada pengeluaran per akun bulan ini"
                textSize = 14f
            }
            layoutAccountSummary.addView(tv)
            return
        }

        accounts.forEach { item ->
            val percent = if (monthTotal > 0) (item.total * 100 / monthTotal) else 0
            val tv = TextView(this).apply {
                text = "${item.accountName} • ${formatRupiah(item.total)} • $percent%"
                textSize = 14f
                setPadding(0, 0, 0, 14)
            }
            layoutAccountSummary.addView(tv)
        }
    }

    private fun buildInsight(
        categoryTotals: List<CategoryTotal>,
        monthTotal: Long,
        transactionCount: Int
    ): String {
        if (monthTotal <= 0L || transactionCount <= 0) {
            return "Belum ada transaksi bulan ini, jadi insight belum bisa dihitung."
        }

        val top = categoryTotals.firstOrNull()
        val avg = monthTotal / transactionCount

        return if (top != null) {
            "Kategori terbesar bulan ini adalah ${top.category} sebesar ${formatRupiah(top.total)}. Rata-rata pengeluaran per transaksi bulan ini ${formatRupiah(avg)}."
        } else {
            "Total bulan ini ${formatRupiah(monthTotal)} dengan $transactionCount transaksi."
        }
    }

    private fun buildBudgetInsight(
        budgets: List<BudgetEntity>,
        categoryTotals: List<CategoryTotal>
    ): String {
        if (budgets.isEmpty()) {
            return "Belum ada budget kategori. Set budget biar app bisa kasih warning."
        }

        val over = budgets.mapNotNull { budget ->
            val spent = categoryTotals.firstOrNull { it.category == budget.category }?.total ?: 0L
            if (spent > budget.amount) {
                "${budget.category} lewat ${formatRupiah(spent - budget.amount)}"
            } else {
                null
            }
        }

        if (over.isNotEmpty()) {
            return "Warning: ${over.joinToString(", ")}"
        }

        val nearest = budgets.mapNotNull { budget ->
            val spent = categoryTotals.firstOrNull { it.category == budget.category }?.total ?: 0L
            if (budget.amount > 0) Pair(budget.category, spent * 100 / budget.amount) else null
        }.maxByOrNull { it.second }

        return if (nearest != null) {
            "Semua budget masih aman. Pemakaian tertinggi sekarang di ${nearest.first} sebesar ${nearest.second}%."
        } else {
            "Semua budget masih aman."
        }
    }

    private suspend fun buildThreeMonthStats(now: Long): String {
        val currentStart = getStartOfMonth(now)
        val nextStart = getStartOfNextMonth(now)

        val prev1Start = getStartOfMonth(addMonths(now, -1))
        val prev2Start = getStartOfMonth(addMonths(now, -2))

        val totalCurrent = transactionDao.getTotalBetween(currentStart, nextStart)
        val totalPrev1 = transactionDao.getTotalBetween(prev1Start, currentStart)
        val totalPrev2 = transactionDao.getTotalBetween(prev2Start, prev1Start)

        val nameCurrent = formatMonthLabel(currentStart)
        val namePrev1 = formatMonthLabel(prev1Start)
        val namePrev2 = formatMonthLabel(prev2Start)

        val trend = when {
            totalCurrent > totalPrev1 -> "naik dibanding bulan lalu"
            totalCurrent < totalPrev1 -> "turun dibanding bulan lalu"
            else -> "sama dengan bulan lalu"
        }

        return "$namePrev2: ${formatRupiah(totalPrev2)}\n$namePrev1: ${formatRupiah(totalPrev1)}\n$nameCurrent: ${formatRupiah(totalCurrent)}\nTren bulan ini $trend."
    }

    private fun showSettingsDialog() {
        val view = LayoutInflater.from(this)
            .inflate(R.layout.dialog_settings, null, false)

        val btnBackupJson = view.findViewById<Button>(R.id.btnBackupJson)
        val btnRestoreJson = view.findViewById<Button>(R.id.btnRestoreJson)
        val btnResetAllData = view.findViewById<Button>(R.id.btnResetAllData)

        val dialog = AlertDialog.Builder(this)
            .setTitle(R.string.title_settings_backup)
            .setView(view)
            .setNegativeButton(R.string.action_close, null)
            .create()

        dialog.show()

        btnBackupJson.setOnClickListener {
            dialog.dismiss()
            createBackupLauncher.launch("walletai_backup_${formatBackupFileName()}.json")
        }

        btnRestoreJson.setOnClickListener {
            dialog.dismiss()
            restoreBackupLauncher.launch(arrayOf("application/json", "text/plain"))
        }

        btnResetAllData.setOnClickListener {
            dialog.dismiss()
            showResetDataDialog()
        }
    }

    private fun showResetDataDialog() {
        AlertDialog.Builder(this)
            .setTitle(R.string.title_reset_data)
            .setMessage(R.string.msg_reset_data_confirm)
            .setNegativeButton(R.string.action_cancel, null)
            .setPositiveButton(R.string.action_delete) { _, _ ->
                lifecycleScope.launch {
                    try {
                        withContext(Dispatchers.IO) {
                            db.clearAllTables()
                        }
                        seedDefaults()
                        loadMasters()
                        refreshAll()
                        Toast.makeText(
                            this@MainActivity,
                            getString(R.string.toast_reset_success),
                            Toast.LENGTH_SHORT
                        ).show()
                    } catch (e: Exception) {
                        Toast.makeText(
                            this@MainActivity,
                            "Reset gagal: ${e.message}",
                            Toast.LENGTH_LONG
                        ).show()
                    }
                }
            }
            .show()
    }

    private suspend fun backupToUri(uri: Uri) {
        withContext(Dispatchers.IO) {
            val transactions = transactionDao.getAllTransactions()
            val accounts = accountDao.getAll()
            val categories = categoryDao.getAll()
            val budgets = budgetDao.getAll()
            val recurring = recurringDao.getAll()

            val root = JSONObject().apply {
                put("exportedAt", System.currentTimeMillis())
                put("transactions", JSONArray().apply {
                    transactions.forEach { put(transactionToJson(it)) }
                })
                put("accounts", JSONArray().apply {
                    accounts.forEach { put(accountToJson(it)) }
                })
                put("categories", JSONArray().apply {
                    categories.forEach { put(categoryToJson(it)) }
                })
                put("budgets", JSONArray().apply {
                    budgets.forEach { put(budgetToJson(it)) }
                })
                put("recurringExpenses", JSONArray().apply {
                    recurring.forEach { put(recurringToJson(it)) }
                })
            }

            contentResolver.openOutputStream(uri)?.bufferedWriter()?.use {
                it.write(root.toString(2))
            } ?: throw IllegalStateException("Output stream tidak tersedia")
        }
    }

    private suspend fun restoreFromUri(uri: Uri) {
        withContext(Dispatchers.IO) {
            val jsonText = contentResolver.openInputStream(uri)?.bufferedReader()?.use {
                it.readText()
            } ?: throw IllegalStateException("Input stream tidak tersedia")

            val root = JSONObject(jsonText)

            val transactions = mutableListOf<TransactionEntity>()
            val accounts = mutableListOf<AccountEntity>()
            val categories = mutableListOf<CategoryEntity>()
            val budgets = mutableListOf<BudgetEntity>()
            val recurring = mutableListOf<RecurringExpenseEntity>()

            val transactionsArray = root.optJSONArray("transactions") ?: JSONArray()
            for (i in 0 until transactionsArray.length()) {
                transactions.add(jsonToTransaction(transactionsArray.getJSONObject(i)))
            }

            val accountsArray = root.optJSONArray("accounts") ?: JSONArray()
            for (i in 0 until accountsArray.length()) {
                accounts.add(jsonToAccount(accountsArray.getJSONObject(i)))
            }

            val categoriesArray = root.optJSONArray("categories") ?: JSONArray()
            for (i in 0 until categoriesArray.length()) {
                categories.add(jsonToCategory(categoriesArray.getJSONObject(i)))
            }

            val budgetsArray = root.optJSONArray("budgets") ?: JSONArray()
            for (i in 0 until budgetsArray.length()) {
                budgets.add(jsonToBudget(budgetsArray.getJSONObject(i)))
            }

            val recurringArray = root.optJSONArray("recurringExpenses") ?: JSONArray()
            for (i in 0 until recurringArray.length()) {
                recurring.add(jsonToRecurring(recurringArray.getJSONObject(i)))
            }

            db.clearAllTables()

            if (categories.isNotEmpty()) {
                categoryDao.insertAll(categories)
            } else {
                seedDefaults()
            }

            if (accounts.isNotEmpty()) {
                accountDao.insertAll(accounts)
            } else {
                seedDefaults()
            }

            budgets.forEach { budgetDao.upsert(it) }
            recurring.forEach { recurringDao.insert(it) }
            transactions.forEach { transactionDao.insert(it) }
        }

        loadMasters()
        refreshAll()
    }

    private fun transactionToJson(item: TransactionEntity): JSONObject {
        return JSONObject().apply {
            put("id", item.id)
            put("merchant", item.merchant)
            put("category", item.category)
            put("accountName", item.accountName)
            put("amount", item.amount)
            put("transactionDate", item.transactionDate)
            put("note", item.note)
            put("receiptPath", item.receiptPath ?: JSONObject.NULL)
            put("rawText", item.rawText ?: JSONObject.NULL)
            put("sourceType", item.sourceType)
            put("recurringId", item.recurringId ?: JSONObject.NULL)
            put("createdAt", item.createdAt)
        }
    }

    private fun accountToJson(item: AccountEntity): JSONObject {
        return JSONObject().apply {
            put("name", item.name)
        }
    }

    private fun categoryToJson(item: CategoryEntity): JSONObject {
        return JSONObject().apply {
            put("name", item.name)
        }
    }

    private fun budgetToJson(item: BudgetEntity): JSONObject {
        return JSONObject().apply {
            put("category", item.category)
            put("amount", item.amount)
        }
    }

    private fun recurringToJson(item: RecurringExpenseEntity): JSONObject {
        return JSONObject().apply {
            put("id", item.id)
            put("merchant", item.merchant)
            put("category", item.category)
            put("accountName", item.accountName)
            put("amount", item.amount)
            put("note", item.note)
            put("dayOfMonth", item.dayOfMonth)
            put("isActive", item.isActive)
            put("createdAt", item.createdAt)
        }
    }

    private fun jsonToTransaction(obj: JSONObject): TransactionEntity {
        return TransactionEntity(
            id = obj.optLong("id", 0L),
            merchant = obj.optString("merchant", "Tanpa Merchant"),
            category = obj.optString("category", "Lainnya"),
            accountName = obj.optString("accountName", "Cash"),
            amount = obj.optLong("amount", 0L),
            transactionDate = obj.optLong("transactionDate", System.currentTimeMillis()),
            note = obj.optString("note", ""),
            receiptPath = obj.optString("receiptPath").takeIf { it.isNotBlank() && it != "null" },
            rawText = obj.optString("rawText").takeIf { it.isNotBlank() && it != "null" },
            sourceType = obj.optString("sourceType", "manual"),
            recurringId = if (obj.isNull("recurringId")) null else obj.optLong("recurringId"),
            createdAt = obj.optLong("createdAt", System.currentTimeMillis())
        )
    }

    private fun jsonToAccount(obj: JSONObject): AccountEntity {
        return AccountEntity(
            name = obj.optString("name", "Cash")
        )
    }

    private fun jsonToCategory(obj: JSONObject): CategoryEntity {
        return CategoryEntity(
            name = obj.optString("name", "Lainnya")
        )
    }

    private fun jsonToBudget(obj: JSONObject): BudgetEntity {
        return BudgetEntity(
            category = obj.optString("category", "Lainnya"),
            amount = obj.optLong("amount", 0L)
        )
    }

    private fun jsonToRecurring(obj: JSONObject): RecurringExpenseEntity {
        return RecurringExpenseEntity(
            id = obj.optLong("id", 0L),
            merchant = obj.optString("merchant", "Tagihan Rutin"),
            category = obj.optString("category", "Lainnya"),
            accountName = obj.optString("accountName", "Cash"),
            amount = obj.optLong("amount", 0L),
            note = obj.optString("note", ""),
            dayOfMonth = obj.optInt("dayOfMonth", 1),
            isActive = obj.optBoolean("isActive", true),
            createdAt = obj.optLong("createdAt", System.currentTimeMillis())
        )
    }

    private fun showBudgetDialog() {
        val view = LayoutInflater.from(this)
            .inflate(R.layout.dialog_budget, null, false)

        val spinnerCategory = view.findViewById<Spinner>(R.id.spinnerBudgetCategory)
        val etBudgetAmount = view.findViewById<EditText>(R.id.etBudgetAmount)

        val categoryAdapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_item,
            categoryNames
        )
        categoryAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        spinnerCategory.adapter = categoryAdapter

        AlertDialog.Builder(this)
            .setTitle(R.string.title_set_budget)
            .setView(view)
            .setNegativeButton(R.string.action_cancel, null)
            .setPositiveButton(R.string.action_save, null)
            .create()
            .also { dialog ->
                dialog.setOnShowListener {
                    dialog.getButton(AlertDialog.BUTTON_POSITIVE).setOnClickListener {
                        val amount = etBudgetAmount.text.toString().trim().toLongOrNull()

                        etBudgetAmount.error = null

                        if (amount == null || amount <= 0L) {
                            etBudgetAmount.error = getString(R.string.error_amount_invalid)
                            etBudgetAmount.requestFocus()
                            return@setOnClickListener
                        }

                        lifecycleScope.launch {
                            val category = spinnerCategory.selectedItem?.toString() ?: "Lainnya"

                            budgetDao.upsert(
                                BudgetEntity(
                                    category = category,
                                    amount = amount
                                )
                            )

                            Toast.makeText(
                                this@MainActivity,
                                getString(R.string.toast_budget_saved),
                                Toast.LENGTH_SHORT
                            ).show()
                            dialog.dismiss()
                            refreshAll()
                        }
                    }
                }
            }
            .show()
    }

    private fun showRecurringDialog() {
        val view = LayoutInflater.from(this)
            .inflate(R.layout.dialog_recurring, null, false)

        val etMerchant = view.findViewById<EditText>(R.id.etRecurringMerchant)
        val etAmount = view.findViewById<EditText>(R.id.etRecurringAmount)
        val spinnerCategory = view.findViewById<Spinner>(R.id.spinnerRecurringCategory)
        val spinnerAccount = view.findViewById<Spinner>(R.id.spinnerRecurringAccount)
        val etDay = view.findViewById<EditText>(R.id.etRecurringDay)
        val etNote = view.findViewById<EditText>(R.id.etRecurringNote)

        val categoryAdapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_item,
            categoryNames
        )
        categoryAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        spinnerCategory.adapter = categoryAdapter

        val accountAdapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_item,
            accountNames
        )
        accountAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        spinnerAccount.adapter = accountAdapter

        AlertDialog.Builder(this)
            .setTitle(R.string.title_recurring)
            .setView(view)
            .setNegativeButton(R.string.action_cancel, null)
            .setPositiveButton(R.string.action_save, null)
            .create()
            .also { dialog ->
                dialog.setOnShowListener {
                    dialog.getButton(AlertDialog.BUTTON_POSITIVE).setOnClickListener {
                        val merchant = etMerchant.text.toString().trim()
                        val amount = etAmount.text.toString().trim().toLongOrNull()
                        val day = etDay.text.toString().trim().toIntOrNull()

                        etMerchant.error = null
                        etAmount.error = null
                        etDay.error = null

                        var valid = true

                        if (merchant.isBlank()) {
                            etMerchant.error = getString(R.string.error_merchant_required)
                            if (valid) etMerchant.requestFocus()
                            valid = false
                        }

                        if (amount == null || amount <= 0L) {
                            etAmount.error = getString(R.string.error_amount_invalid)
                            if (valid) etAmount.requestFocus()
                            valid = false
                        }

                        if (day == null || day !in 1..31) {
                            etDay.error = getString(R.string.error_day_invalid)
                            if (valid) etDay.requestFocus()
                            valid = false
                        }

                        if (!valid) return@setOnClickListener

                        lifecycleScope.launch {
                            recurringDao.insert(
                                RecurringExpenseEntity(
                                    merchant = merchant,
                                    amount = amount ?: 0L,
                                    category = spinnerCategory.selectedItem?.toString() ?: "Lainnya",
                                    accountName = spinnerAccount.selectedItem?.toString() ?: "Cash",
                                    note = etNote.text.toString().trim(),
                                    dayOfMonth = day ?: 1
                                )
                            )

                            Toast.makeText(
                                this@MainActivity,
                                getString(R.string.toast_recurring_saved),
                                Toast.LENGTH_SHORT
                            ).show()
                            dialog.dismiss()
                            refreshAll()
                        }
                    }
                }
            }
            .show()
    }

    private fun showTransactionDialog(
        existing: TransactionEntity? = null,
        merchant: String = existing?.merchant ?: "",
        amount: Long = existing?.amount ?: 0L,
        category: String = existing?.category ?: "Makan",
        accountName: String = existing?.accountName ?: "Cash",
        note: String = existing?.note ?: "",
        receiptPath: String? = existing?.receiptPath,
        rawText: String? = existing?.rawText,
        sourceType: String = existing?.sourceType ?: "manual",
        transactionDate: Long = existing?.transactionDate ?: System.currentTimeMillis()
    ) {
        val view = LayoutInflater.from(this)
            .inflate(R.layout.dialog_add_transaction, null, false)

        val etMerchant = view.findViewById<EditText>(R.id.etMerchant)
        val etAmount = view.findViewById<EditText>(R.id.etAmount)
        val btnPickDate = view.findViewById<Button>(R.id.btnPickDate)
        val tvSelectedDate = view.findViewById<TextView>(R.id.tvSelectedDate)
        val spinnerCategory = view.findViewById<Spinner>(R.id.spinnerCategory)
        val spinnerAccount = view.findViewById<Spinner>(R.id.spinnerAccount)
        val etNote = view.findViewById<EditText>(R.id.etNote)

        var selectedDateMillis = transactionDate

        etMerchant.setText(merchant)
        if (amount > 0L) etAmount.setText(amount.toString())
        etNote.setText(note)
        tvSelectedDate.text = formatDateOnly(selectedDateMillis)

        btnPickDate.setOnClickListener {
            val cal = Calendar.getInstance().apply { timeInMillis = selectedDateMillis }

            DatePickerDialog(
                this,
                { _, year, month, dayOfMonth ->
                    val newCal = Calendar.getInstance().apply { timeInMillis = selectedDateMillis }
                    newCal.set(Calendar.YEAR, year)
                    newCal.set(Calendar.MONTH, month)
                    newCal.set(Calendar.DAY_OF_MONTH, dayOfMonth)
                    selectedDateMillis = newCal.timeInMillis
                    tvSelectedDate.text = formatDateOnly(selectedDateMillis)
                },
                cal.get(Calendar.YEAR),
                cal.get(Calendar.MONTH),
                cal.get(Calendar.DAY_OF_MONTH)
            ).show()
        }

        val categoryAdapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_item,
            categoryNames
        )
        categoryAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        spinnerCategory.adapter = categoryAdapter

        val accountAdapter = ArrayAdapter(
            this,
            android.R.layout.simple_spinner_item,
            accountNames
        )
        accountAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        spinnerAccount.adapter = accountAdapter

        val categoryIndex = categoryNames.indexOf(category).takeIf { it >= 0 } ?: 0
        val accountIndex = accountNames.indexOf(accountName).takeIf { it >= 0 } ?: 0
        spinnerCategory.setSelection(categoryIndex)
        spinnerAccount.setSelection(accountIndex)

        AlertDialog.Builder(this)
            .setTitle(if (existing == null) R.string.title_add_transaction else R.string.title_edit_transaction)
            .setView(view)
            .setNegativeButton(R.string.action_cancel, null)
            .setPositiveButton(R.string.action_save, null)
            .create()
            .also { dialog ->
                dialog.setOnShowListener {
                    dialog.getButton(AlertDialog.BUTTON_POSITIVE).setOnClickListener {
                        val merchantValue = etMerchant.text.toString().trim()
                        val amountValue = etAmount.text.toString().trim().toLongOrNull()

                        etMerchant.error = null
                        etAmount.error = null

                        var valid = true

                        if (merchantValue.isBlank()) {
                            etMerchant.error = getString(R.string.error_merchant_required)
                            etMerchant.requestFocus()
                            valid = false
                        }

                        if (amountValue == null || amountValue <= 0L) {
                            etAmount.error = getString(R.string.error_amount_invalid)
                            if (valid) etAmount.requestFocus()
                            valid = false
                        }

                        if (!valid) return@setOnClickListener

                        lifecycleScope.launch {
                            val item = TransactionEntity(
                                id = existing?.id ?: 0L,
                                merchant = merchantValue,
                                amount = amountValue ?: 0L,
                                category = spinnerCategory.selectedItem?.toString() ?: "Lainnya",
                                accountName = spinnerAccount.selectedItem?.toString() ?: "Cash",
                                note = etNote.text.toString().trim(),
                                transactionDate = selectedDateMillis,
                                receiptPath = receiptPath,
                                rawText = rawText,
                                sourceType = sourceType,
                                recurringId = existing?.recurringId,
                                createdAt = existing?.createdAt ?: System.currentTimeMillis()
                            )

                            if (existing == null) {
                                transactionDao.insert(item)
                                Toast.makeText(
                                    this@MainActivity,
                                    getString(R.string.toast_transaction_saved),
                                    Toast.LENGTH_SHORT
                                ).show()
                            } else {
                                transactionDao.update(item)
                                Toast.makeText(
                                    this@MainActivity,
                                    getString(R.string.toast_transaction_updated),
                                    Toast.LENGTH_SHORT
                                ).show()
                            }

                            dialog.dismiss()
                            refreshAll()
                        }
                    }
                }
            }
            .show()
    }

    private fun handleGalleryImage(uri: Uri) {
        try {
            val copiedFile = copyUriToInternalFile(uri)
            if (copiedFile == null) {
                Toast.makeText(
                    this,
                    getString(R.string.toast_gallery_read_failed),
                    Toast.LENGTH_SHORT
                ).show()
                return
            }

            val image = InputImage.fromFilePath(this, uri)
            val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

            recognizer.process(image)
                .addOnSuccessListener { result ->
                    val parsed = parseReceipt(result.text)
                    val suggestedCategory = suggestCategoryFromOcr(parsed.merchant, result.text)

                    showTransactionDialog(
                        existing = null,
                        merchant = parsed.merchant,
                        amount = parsed.amount,
                        category = suggestedCategory,
                        accountName = "Cash",
                        note = "",
                        receiptPath = copiedFile.absolutePath,
                        rawText = result.text,
                        sourceType = "gallery",
                        transactionDate = System.currentTimeMillis()
                    )
                }
                .addOnFailureListener { e ->
                    Toast.makeText(this, "OCR galeri gagal: ${e.message}", Toast.LENGTH_SHORT).show()
                }
        } catch (e: Exception) {
            Toast.makeText(this, "Gagal buka gambar: ${e.message}", Toast.LENGTH_SHORT).show()
        }
    }

    private fun copyUriToInternalFile(uri: Uri): File? {
        val receiptsDir = File(filesDir, "gallery_receipts")
        if (!receiptsDir.exists()) receiptsDir.mkdirs()

        val outFile = File(receiptsDir, "gallery_${System.currentTimeMillis()}.jpg")

        return try {
            val input = contentResolver.openInputStream(uri) ?: return null
            input.use { stream ->
                outFile.outputStream().use { output ->
                    stream.copyTo(output)
                }
            }
            outFile
        } catch (e: Exception) {
            null
        }
    }

    private fun parseReceipt(raw: String): OcrParseResult {
        val lines = raw.lines()
            .map { it.trim() }
            .filter { it.isNotBlank() }

        val merchant = lines.firstOrNull()?.take(40) ?: "Hasil Scan"

        val totalLine = lines.firstOrNull {
            val lower = it.lowercase(Locale.getDefault())
            lower.contains("total") || lower.contains("grand total")
        }

        val regex = Regex("""(?:rp\s*)?[\d.,]{3,}""", RegexOption.IGNORE_CASE)

        val candidates = buildList {
            totalLine?.let { addAll(regex.findAll(it).map { m -> m.value }) }
            addAll(regex.findAll(raw).map { it.value })
        }.map { text ->
            text.lowercase(Locale.getDefault())
                .replace("rp", "")
                .replace(".", "")
                .replace(",", "")
                .replace(" ", "")
                .trim()
        }.mapNotNull { it.toLongOrNull() }
            .filter { it > 0 }
            .toList()

        val amount = candidates.maxOrNull() ?: 0L

        return OcrParseResult(
            merchant = merchant,
            amount = amount
        )
    }

    private fun suggestCategoryFromOcr(
        merchant: String,
        rawText: String?
    ): String {
        val text = "${merchant.lowercase(Locale.getDefault())} ${rawText.orEmpty().lowercase(Locale.getDefault())}"

        return when {
            containsAny(text, listOf("kopi", "coffee", "cafe", "resto", "restaurant", "bakmi", "nasi", "ayam", "mcd", "kfc", "burger", "pizza", "starbucks", "chatime")) -> "Makan"
            containsAny(text, listOf("grab", "gocar", "gojek", "taxi", "transjakarta", "mrt", "shell", "pertamina", "spbu", "toll", "parkir")) -> "Transport"
            containsAny(text, listOf("tokopedia", "shopee", "blibli", "lazada", "matahari", "uniqlo", "h&m", "miniso", "alfamart", "indomaret")) -> "Belanja"
            containsAny(text, listOf("pln", "pdam", "telkom", "indihome", "xl", "telkomsel", "smartfren", "by.u", "internet", "listrik", "air")) -> "Tagihan"
            containsAny(text, listOf("cinema", "xxi", "cgv", "netflix", "spotify", "steam", "playstation")) -> "Hiburan"
            containsAny(text, listOf("apotek", "kimia farma", "guardian", "halodoc", "klinik", "hospital", "rs", "dokter")) -> "Kesehatan"
            else -> "Lainnya"
        }
    }

    private fun containsAny(text: String, keywords: List<String>): Boolean {
        return keywords.any { text.contains(it) }
    }

    private fun exportCurrentCsv() {
        if (currentVisibleItems.isEmpty()) {
            Toast.makeText(
                this,
                getString(R.string.toast_no_export_data),
                Toast.LENGTH_SHORT
            ).show()
            return
        }

        try {
            val exportDir = File(filesDir, "exports")
            if (!exportDir.exists()) exportDir.mkdirs()

            val file = File(exportDir, "walletai_export_${System.currentTimeMillis()}.csv")

            val csv = buildString {
                appendLine("id,merchant,category,account,amount,date,note,sourceType,receiptPath")
                currentVisibleItems.forEach { item ->
                    appendLine(
                        listOf(
                            item.id.toString(),
                            escapeCsv(item.merchant),
                            escapeCsv(item.category),
                            escapeCsv(item.accountName),
                            item.amount.toString(),
                            escapeCsv(formatDate(item.transactionDate)),
                            escapeCsv(item.note),
                            escapeCsv(item.sourceType),
                            escapeCsv(item.receiptPath.orEmpty())
                        ).joinToString(",")
                    )
                }
            }

            file.writeText(csv)

            val uri: Uri = FileProvider.getUriForFile(
                this,
                "${packageName}.fileprovider",
                file
            )

            val intent = Intent(Intent.ACTION_SEND).apply {
                type = "text/csv"
                putExtra(Intent.EXTRA_STREAM, uri)
                putExtra(Intent.EXTRA_SUBJECT, "WalletAI Export")
                addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
            }

            startActivity(Intent.createChooser(intent, "Share CSV"))
        } catch (e: Exception) {
            Toast.makeText(this, "Export gagal: ${e.message}", Toast.LENGTH_LONG).show()
        }
    }

    private fun escapeCsv(value: String): String {
        val safe = value.replace("\"", "\"\"")
        return "\"$safe\""
    }

    private fun showDetailDialog(item: TransactionEntity) {
        val view = LayoutInflater.from(this)
            .inflate(R.layout.dialog_transaction_detail, null, false)

        val imgReceipt = view.findViewById<ImageView>(R.id.imgReceipt)
        val tvDetailMerchant = view.findViewById<TextView>(R.id.tvDetailMerchant)
        val tvDetailAmount = view.findViewById<TextView>(R.id.tvDetailAmount)
        val tvDetailCategory = view.findViewById<TextView>(R.id.tvDetailCategory)
        val tvDetailAccount = view.findViewById<TextView>(R.id.tvDetailAccount)
        val tvDetailDate = view.findViewById<TextView>(R.id.tvDetailDate)
        val tvDetailSource = view.findViewById<TextView>(R.id.tvDetailSource)
        val tvDetailNote = view.findViewById<TextView>(R.id.tvDetailNote)
        val tvDetailRawTextLabel = view.findViewById<TextView>(R.id.tvDetailRawTextLabel)
        val tvDetailRawText = view.findViewById<TextView>(R.id.tvDetailRawText)
        val btnDetailEdit = view.findViewById<Button>(R.id.btnDetailEdit)
        val btnDetailDelete = view.findViewById<Button>(R.id.btnDetailDelete)

        tvDetailMerchant.text = item.merchant
        tvDetailAmount.text = formatRupiah(item.amount)
        tvDetailCategory.text = "Kategori: ${item.category}"
        tvDetailAccount.text = "Akun: ${item.accountName}"
        tvDetailDate.text = "Tanggal: ${formatDate(item.transactionDate)}"
        tvDetailSource.text = "Sumber: ${item.sourceType}"
        tvDetailNote.text = "Catatan: ${if (item.note.isBlank()) "-" else item.note}"

        if (!item.rawText.isNullOrBlank()) {
            tvDetailRawTextLabel.visibility = View.VISIBLE
            tvDetailRawText.visibility = View.VISIBLE
            tvDetailRawText.text = item.rawText
        }

        if (!item.receiptPath.isNullOrBlank()) {
            val file = File(item.receiptPath)
            if (file.exists()) {
                val bitmap = BitmapFactory.decodeFile(file.absolutePath)
                if (bitmap != null) {
                    imgReceipt.visibility = View.VISIBLE
                    imgReceipt.setImageBitmap(bitmap)
                }
            }
        }

        val dialog = AlertDialog.Builder(this)
            .setTitle(R.string.title_transaction_detail)
            .setView(view)
            .setNegativeButton(R.string.action_close, null)
            .create()

        dialog.show()

        btnDetailEdit.setOnClickListener {
            dialog.dismiss()
            showTransactionDialog(existing = item)
        }

        btnDetailDelete.setOnClickListener {
            dialog.dismiss()
            showDeleteDialog(item)
        }
    }

    private fun showDeleteDialog(item: TransactionEntity) {
        AlertDialog.Builder(this)
            .setTitle(R.string.title_delete_transaction)
            .setMessage(getString(R.string.msg_delete_transaction, item.merchant))
            .setNegativeButton(R.string.action_cancel, null)
            .setPositiveButton(R.string.action_delete) { _, _ ->
                lifecycleScope.launch {
                    transactionDao.delete(item)
                    refreshAll()
                    Toast.makeText(
                        this@MainActivity,
                        getString(R.string.toast_transaction_deleted),
                        Toast.LENGTH_SHORT
                    ).show()
                }
            }
            .show()
    }

    private fun getStartOfWeek(now: Long): Long {
        val cal = Calendar.getInstance().apply { timeInMillis = now }
        cal.firstDayOfWeek = Calendar.MONDAY
        cal.set(Calendar.DAY_OF_WEEK, Calendar.MONDAY)
        cal.set(Calendar.HOUR_OF_DAY, 0)
        cal.set(Calendar.MINUTE, 0)
        cal.set(Calendar.SECOND, 0)
        cal.set(Calendar.MILLISECOND, 0)
        return cal.timeInMillis
    }

    private fun getStartOfMonth(now: Long): Long {
        val cal = Calendar.getInstance().apply { timeInMillis = now }
        cal.set(Calendar.DAY_OF_MONTH, 1)
        cal.set(Calendar.HOUR_OF_DAY, 0)
        cal.set(Calendar.MINUTE, 0)
        cal.set(Calendar.SECOND, 0)
        cal.set(Calendar.MILLISECOND, 0)
        return cal.timeInMillis
    }

    private fun getStartOfNextMonth(now: Long): Long {
        val cal = Calendar.getInstance().apply { timeInMillis = now }
        cal.set(Calendar.DAY_OF_MONTH, 1)
        cal.add(Calendar.MONTH, 1)
        cal.set(Calendar.HOUR_OF_DAY, 0)
        cal.set(Calendar.MINUTE, 0)
        cal.set(Calendar.SECOND, 0)
        cal.set(Calendar.MILLISECOND, 0)
        return cal.timeInMillis
    }

    private fun addMonths(now: Long, amount: Int): Long {
        val cal = Calendar.getInstance().apply { timeInMillis = now }
        cal.add(Calendar.MONTH, amount)
        return cal.timeInMillis
    }

    private fun formatMonthLabel(value: Long): String {
        return SimpleDateFormat("MMM yyyy", Locale("id", "ID"))
            .format(Date(value))
    }

    private fun formatRupiah(value: Long): String {
        val format = NumberFormat.getCurrencyInstance(Locale("id", "ID"))
        return format.format(value).replace(",00", "")
    }

    private fun formatDate(value: Long): String {
        return SimpleDateFormat("dd MMM yyyy, HH:mm", Locale("id", "ID"))
            .format(Date(value))
    }

    private fun formatDateOnly(value: Long): String {
        return SimpleDateFormat("dd MMM yyyy", Locale("id", "ID"))
            .format(Date(value))
    }

    private fun formatBackupFileName(): String {
        return SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault())
            .format(Date())
    }

    data class OcrParseResult(
        val merchant: String,
        val amount: Long
    )
}

```
