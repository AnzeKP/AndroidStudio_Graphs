package com.example.dat;

import android.graphics.Color;
import android.graphics.drawable.GradientDrawable;
import android.os.Bundle;
import android.view.MenuItem;
import android.widget.TextView;
import android.widget.Toast;
import androidx.annotation.NonNull;
import androidx.appcompat.app.ActionBarDrawerToggle;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.view.GravityCompat;
import androidx.drawerlayout.widget.DrawerLayout;
import com.github.mikephil.charting.charts.LineChart;
import com.github.mikephil.charting.charts.PieChart;
import com.github.mikephil.charting.components.YAxis;
import com.github.mikephil.charting.data.*;
import com.github.mikephil.charting.utils.ColorTemplate;
import com.google.android.material.navigation.NavigationView;
import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;
import okhttp3.*;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.*;

public class MainActivity extends AppCompatActivity implements NavigationView.OnNavigationItemSelectedListener {
    private LineChart lineChart;
    private PieChart pieChart;
    private DrawerLayout drawerLayout;
    private TextView tvSumCena; // TextView to display the sum of cena
    private static final String URL = " "; // Replace with your API URL
    private static final OkHttpClient client = new OkHttpClient();
    private static final SimpleDateFormat INPUT_FORMAT = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", Locale.getDefault());
    private static final SimpleDateFormat OUTPUT_FORMAT = new SimpleDateFormat("dd/MM", Locale.getDefault());
    private static final int[] COLORS = {Color.BLUE, Color.RED, Color.GREEN, Color.MAGENTA, Color.CYAN, Color.YELLOW};

    static {
        INPUT_FORMAT.setTimeZone(TimeZone.getTimeZone("UTC"));
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Set up the toolbar and hamburger menu
        setSupportActionBar(findViewById(R.id.toolbar));
        drawerLayout = findViewById(R.id.drawer_layout);
        ActionBarDrawerToggle toggle = new ActionBarDrawerToggle(
                this, drawerLayout, findViewById(R.id.toolbar),
                R.string.navigation_drawer_open, R.string.navigation_drawer_close
        );
        drawerLayout.addDrawerListener(toggle);
        toggle.syncState();

        // Set up navigation view
        NavigationView navigationView = findViewById(R.id.nav_view);
        navigationView.setNavigationItemSelectedListener(this);

        // Initialize charts and TextView
        lineChart = findViewById(R.id.chart);
        pieChart = findViewById(R.id.pie_chart);
        tvSumCena = findViewById(R.id.tvSumCena); // Initialize the TextView
        setupCharts();
        fetchData();
    }

    private void setupCharts() {
        // Line chart setup
        lineChart.getDescription().setEnabled(false);
        lineChart.getXAxis().setEnabled(false);
        YAxis yAxis = lineChart.getAxisLeft();
        yAxis.setTextSize(12f);
        yAxis.setTextColor(Color.BLACK);
        yAxis.setAxisLineColor(Color.BLACK);
        yAxis.setDrawGridLines(true);
        yAxis.setGridColor(Color.LTGRAY);
        lineChart.getAxisRight().setEnabled(false);
        lineChart.setBackgroundColor(0xFFE6F0FF);

        // Pie chart setup
        pieChart.getDescription().setEnabled(false);
        pieChart.setBackgroundColor(0xFFE6F0FF);
    }

    @Override
    public boolean onNavigationItemSelected(@NonNull MenuItem item) {
        boolean isLineChart = item.getItemId() == R.id.nav_line_chart;
        lineChart.setVisibility(isLineChart ? LineChart.VISIBLE : LineChart.GONE);
        pieChart.setVisibility(isLineChart ? PieChart.GONE : PieChart.VISIBLE);
        fetchData();
        drawerLayout.closeDrawer(GravityCompat.START);
        return true;
    }

    @Override
    public void onBackPressed() {
        if (drawerLayout.isDrawerOpen(GravityCompat.START)) {
            drawerLayout.closeDrawer(GravityCompat.START);
        } else {
            super.onBackPressed();
        }
    }

    private void fetchData() {
        client.newCall(new Request.Builder().url(URL).build()).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                runOnUiThread(() -> Toast.makeText(MainActivity.this, "Failed to load data", Toast.LENGTH_SHORT).show());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (!response.isSuccessful()) return;

                List<Measurement> measurements = new Gson().fromJson(
                        response.body().string(),
                        new TypeToken<List<Measurement>>(){}.getType()
                );

                runOnUiThread(() -> {
                    if (measurements == null || measurements.isEmpty()) {
                        Toast.makeText(MainActivity.this, "No data", Toast.LENGTH_SHORT).show();
                        return;
                    }

                    // Calculate the sum of cena
                    float sumCena = 0;
                    for (Measurement m : measurements) {
                        sumCena += m.getCena();
                    }
                    tvSumCena.setText(String.format(Locale.getDefault(), "Sum: %.2f", sumCena)); // Update the TextView

                    if (lineChart.getVisibility() == LineChart.VISIBLE) updateLineChart(measurements);
                    else updatePieChart(measurements);
                });
            }
        });
    }

    private void updateLineChart(List<Measurement> measurements) {
        Map<String, List<Entry>> dayMap = new HashMap<>();
        for (int i = 0; i < measurements.size(); i++) {
            Measurement m = measurements.get(i);
            String day = parseDay(m.getCas());
            dayMap.computeIfAbsent(day, k -> new ArrayList<>()).add(new Entry(i, m.getNapetost()));
        }

        LineData data = new LineData();
        int colorIndex = 0;
        for (Map.Entry<String, List<Entry>> e : dayMap.entrySet()) {
            int color = COLORS[colorIndex % COLORS.length];
            LineDataSet set = new LineDataSet(e.getValue(), e.getKey());
            set.setColor(color); // Line color matches gradient
            set.setLineWidth(2f);
            set.setDrawCircles(false);
            set.setDrawFilled(true);
            set.setFillDrawable(createGradient(color)); // Gradient matches line color
            data.addDataSet(set);
            colorIndex++;
        }
        lineChart.setData(data);
        lineChart.invalidate();
    }

    private void updatePieChart(List<Measurement> measurements) {
        Map<String, Float> dayMap = new HashMap<>();
        for (Measurement m : measurements) {
            String day = parseDay(m.getCas());
            dayMap.put(day, dayMap.getOrDefault(day, 0f) + m.getCena());
        }
        PieDataSet set = new PieDataSet(new ArrayList<>(), "Cena by Day");
        for (Map.Entry<String, Float> e : dayMap.entrySet()) {
            if (e.getValue() == 0) continue;
            set.addEntry(new PieEntry(e.getValue(), e.getKey()));
        }
        set.setColors(ColorTemplate.COLORFUL_COLORS);
        set.setValueTextSize(12f);
        set.setValueTextColor(Color.BLACK);
        pieChart.setData(new PieData(set));
        pieChart.invalidate();
    }

    private GradientDrawable createGradient(int color) {
        GradientDrawable gd = new GradientDrawable();
        gd.setColors(new int[]{color, Color.argb(50, Color.red(color), Color.green(color), Color.blue(color))}); // Semi-transparent gradient
        gd.setOrientation(GradientDrawable.Orientation.TOP_BOTTOM);
        return gd;
    }

    private String parseDay(String isoTime) {
        try {
            return OUTPUT_FORMAT.format(INPUT_FORMAT.parse(isoTime));
        } catch (Exception e) {
            return "N/A";
        }
    }
}
