package com.example.nammasantheledger

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.room.*
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch

// --------------------
// ROOM ENTITY
// --------------------

@Entity(tableName = "customers")
data class Customer(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val name: String,
    val phone: String,
    val balance: Double = 0.0
)

@Entity(
    tableName = "transactions",
    foreignKeys = [
        ForeignKey(
            entity = Customer::class,
            parentColumns = ["id"],
            childColumns = ["customerId"],
            onDelete = ForeignKey.CASCADE
        )
    ]
)
data class Transaction(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val customerId: Int,
    val amount: Double,
    val type: String
)

// --------------------
// DAO
// --------------------

@Dao
interface LedgerDao {

    @Insert
    suspend fun insertCustomer(customer: Customer)

    @Insert
    suspend fun insertTransaction(transaction: Transaction)

    @Query("SELECT * FROM customers")
    fun getCustomers(): Flow<List<Customer>>

    @Query("UPDATE customers SET balance = balance + :amount WHERE id = :customerId")
    suspend fun updateBalance(customerId: Int, amount: Double)
}

// --------------------
// DATABASE
// --------------------

@Database(
    entities = [Customer::class, Transaction::class],
    version = 1
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun dao(): LedgerDao
}

// --------------------
// VIEWMODEL
// --------------------

class LedgerViewModel(private val dao: LedgerDao) : ViewModel() {

    var customers by mutableStateOf<List<Customer>>(emptyList())

    init {
        viewModelScope.launch {
            dao.getCustomers().collectLatest {
                customers = it
            }
        }
    }

    fun addCustomer(name: String, phone: String) {
        viewModelScope.launch {
            dao.insertCustomer(
                Customer(
                    name = name,
                    phone = phone
                )
            )
        }
    }

    fun addCredit(customerId: Int, amount: Double) {
        viewModelScope.launch {

            dao.insertTransaction(
                Transaction(
                    customerId = customerId,
                    amount = amount,
                    type = "Credit"
                )
            )

            dao.updateBalance(customerId, amount)
        }
    }

    fun addPayment(customerId: Int, amount: Double) {
        viewModelScope.launch {

            dao.insertTransaction(
                Transaction(
                    customerId = customerId,
                    amount = amount,
                    type = "Payment"
                )
            )

            dao.updateBalance(customerId, -amount)
        }
    }
}

// --------------------
// MAIN ACTIVITY
// --------------------

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val db = Room.databaseBuilder(
            applicationContext,
            AppDatabase::class.java,
            "ledger_db"
        ).build()

        val viewModel = LedgerViewModel(db.dao())

        setContent {
            MaterialTheme {
                LedgerScreen(viewModel)
            }
        }
    }
}

// --------------------
// UI SCREEN
// --------------------

@Composable
fun LedgerScreen(viewModel: LedgerViewModel) {

    var name by remember { mutableStateOf("") }
    var phone by remember { mutableStateOf("") }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {

        Text(
            text = "Namma Santhe Ledger",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("Customer Name") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        OutlinedTextField(
            value = phone,
            onValueChange = { phone = it },
            label = { Text("Phone Number") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = {
                if (name.isNotEmpty()) {
                    viewModel.addCustomer(name, phone)
                    name = ""
                    phone = ""
                }
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Add Customer")
        }

        Spacer(modifier = Modifier.height(20.dp))

        if (viewModel.customers.isEmpty()) {

            Text("No customers added yet.")

        } else {

            LazyColumn {

                items(viewModel.customers) { customer ->

                    Card(
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(vertical = 8.dp)
                    ) {

                        Column(
                            modifier = Modifier.padding(16.dp)
                        ) {

                            Text(
                                text = customer.name,
                                style = MaterialTheme.typography.titleMedium
                            )

                            Text(text = customer.phone)

                            Spacer(modifier = Modifier.height(4.dp))

                            Text(
                                text = "Balance: ₹${customer.balance}"
                            )

                            Spacer(modifier = Modifier.height(12.dp))

                            Row {

                                Button(
                                    onClick = {
                                        viewModel.addCredit(customer.id, 100.0)
                                    }
                                ) {
                                    Text("Add Credit")
                                }

                                Spacer(modifier = Modifier.width(8.dp))

                                Button(
                                    onClick = {
                                        viewModel.addPayment(customer.id, 50.0)
                                    }
                                ) {
                                    Text("Add Payment")
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
