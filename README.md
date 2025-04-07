package com.example.algotrader

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import android.widget.*
import androidx.lifecycle.lifecycleScope
import com.github.mikephil.charting.charts.LineChart
import com.github.mikephil.charting.data.*
import com.github.mikephil.charting.components.XAxis
import kotlinx.coroutines.*
import kotlin.random.Random

data class StockData(
    val date: String,
    val open: Double,
    val high: Double,
    val low: Double,
    val close: Double,
    val volume: Long
)

class MainActivity : AppCompatActivity() {

    private lateinit var chart: LineChart
    private lateinit var tradeCallText: TextView
    private lateinit var supportResText: TextView
    private lateinit var volumeText: TextView
    private lateinit var symbolInput: EditText
    private lateinit var fetchButton: Button

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(makeUI())

        fetchButton.setOnClickListener {
            val symbol = symbolInput.text.toString().uppercase()
            lifecycleScope.launch {
                val data = fetchMockData(symbol)
                displayChart(data)
                supportResText.text = getSupportResistance(data)
                tradeCallText.text = generateTradeCall(data)
                volumeText.text = "📦 Volume: ${data.last().volume / 1_000_000}M"
            }
        }
    }

    private fun getSupportResistance(data: List<StockData>): String {
        val lows = data.map { it.low }
        val highs = data.map { it.high }
        val support = lows.minOrNull() ?: 0.0
        val resistance = highs.maxOrNull() ?: 0.0
        return "🔻 Support: $support | 🔺 Resistance: $resistance"
    }

    private fun generateTradeCall(data: List<StockData>): String {
        if (data.isEmpty()) return "⚠️ No data"
        val latest = data.last()
        val lows = data.map { it.low }
        val highs = data.map { it.high }
        val support = lows.minOrNull() ?: 0.0
        val resistance = highs.maxOrNull() ?: 0.0

        return when {
            latest.close <= support * 1.02 -> "🔔 Buy at ${latest.close} (🎯 Target: $resistance, 🛑 SL: ${support * 0.98})"
            latest.close >= resistance * 0.98 -> "🔔 Sell at ${latest.close} (🎯 Target: $support, 🛑 SL: ${resistance * 1.02})"
            else -> "⚠️ No Clear Trade Signal"
        }
    }

    private fun displayChart(data: List<StockData>) {
        val entries = data.mapIndexed { index, item ->
            Entry(index.toFloat(), item.close.toFloat())
        }

        val dataSet = LineDataSet(entries, "Price").apply {
            setDrawValues(false)
            setDrawCircles(false)
            color = resources.getColor(android.R.color.holo_blue_dark)
        }

        chart.data = LineData(dataSet)
        chart.xAxis.position = XAxis.XAxisPosition.BOTTOM
        chart.invalidate()
    }

    // MOCK data fetch
    private suspend fun fetchMockData(symbol: String): List<StockData> = withContext(Dispatchers.IO) {
        delay(1000)
        val mockList = mutableListOf<StockData>()
        var basePrice = Random.nextDouble(100.0, 200.0)
        repeat(30) {
            val open = basePrice + Random.nextDouble(-2.0, 2.0)
            val close = open + Random.nextDouble(-3.0, 3.0)
            val high = maxOf(open, close) + Random.nextDouble(0.5, 2.0)
            val low = minOf(open, close) - Random.nextDouble(0.5, 2.0)
            val volume = Random.nextLong(10_000_000, 50_000_000)
            mockList.add(StockData("2025-04-${it + 1}", open, high, low, close, volume))
            basePrice = close
        }
        mockList
    }

    // Simple UI in code
    private fun makeUI(): LinearLayout {
        val layout = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(24, 24, 24, 24)
        }

        symbolInput = EditText(this).apply {
            hint = "Enter Stock Symbol (e.g. AAPL)"
        }

        fetchButton = Button(this).apply {
            text = "Fetch Data"
        }

        supportResText = TextView(this)
        tradeCallText = TextView(this)
        volumeText = TextView(this)

        chart = LineChart(this)

        layout.addView(symbolInput)
        layout.addView(fetchButton)
        layout.addView(supportResText)
        layout.addView(tradeCallText)
        layout.addView(volumeText)
        layout.addView(chart)

        return layout
    }
}

