
package com.wahyu.ai;

import android.app.Activity;
import android.os.Bundle;
import android.graphics.Color;
import android.graphics.Typeface;
import android.view.Gravity;
import android.view.View;
import android.view.inputmethod.InputMethodManager;
import android.content.Context;
import android.widget.*;
import org.json.JSONArray;
import org.json.JSONObject;
import java.net.URL;
import java.net.HttpURLConnection;
import java.io.OutputStream;
import java.io.InputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class MainActivity extends Activity {

    private LinearLayout root;
    private LinearLayout messages;
    private EditText questionInput;
    private EditText keyInput;
    private Button sendButton;
    private TextView status;

    private final JSONArray history = new JSONArray();
    private boolean loading = false;

    private int blue = Color.rgb(45, 110, 245);
    private int background = Color.rgb(12, 20, 40);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        buildUI();
    }

    private void buildUI() {
        root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(16, 12, 16, 12);
        root.setBackgroundColor(background);

        TextView title = new TextView(this);
        title.setText("WAHYU AI");
        title.setTextColor(Color.CYAN);
        title.setTextSize(26);
        title.setTypeface(null, Typeface.BOLD);
        title.setGravity(Gravity.CENTER);
        root.addView(title);

        TextView subtitle = new TextView(this);
        subtitle.setText("AI Chat Assistant");
        subtitle.setTextColor(Color.LTGRAY);
        subtitle.setGravity(Gravity.CENTER);
        root.addView(subtitle);

        keyInput = new EditText(this);
        keyInput.setSingleLine(true);
        keyInput.setTextColor(Color.WHITE);
        keyInput.setHintTextColor(Color.LTGRAY);
        keyInput.setHint("Masukkan API key untuk tes pribadi");
        keyInput.setInputType(
            android.text.InputType.TYPE_CLASS_TEXT |
            android.text.InputType.TYPE_TEXT_VARIATION_PASSWORD
        );
        root.addView(keyInput);

        ScrollView scroll = new ScrollView(this);
        messages = new LinearLayout(this);
        messages.setOrientation(LinearLayout.VERTICAL);
        messages.setPadding(4, 12, 4, 12);
        scroll.addView(messages);

        root.addView(scroll, new LinearLayout.LayoutParams(
            -1, 0, 1
        ));

        status = new TextView(this);
        status.setTextColor(Color.LTGRAY);
        status.setTextSize(12);
        status.setText("Siap menjawab");
        root.addView(status);

        LinearLayout bottom = new LinearLayout(this);
        bottom.setOrientation(LinearLayout.HORIZONTAL);
        bottom.setGravity(Gravity.CENTER_VERTICAL);

        questionInput = new EditText(this);
        questionInput.setSingleLine(true);
        questionInput.setTextColor(Color.WHITE);
        questionInput.setHintTextColor(Color.LTGRAY);
        questionInput.setHint("Tulis pertanyaan...");
        bottom.addView(questionInput,
            new LinearLayout.LayoutParams(0, -2, 1));

        sendButton = new Button(this);
        sendButton.setText("KIRIM");
        sendButton.setTextColor(Color.WHITE);
        sendButton.setBackgroundTintList(
            android.content.res.ColorStateList.valueOf(blue)
        );
        bottom.addView(sendButton);

        root.addView(bottom);
        setContentView(root);

        addMessage("AI", "Halo! Saya WAHYU AI. Silakan bertanya.");
        sendButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                sendMessage();
            }
        });
    }

    private void addMessage(String who, String text) {
        TextView bubble = new TextView(this);
        bubble.setText(who + ":\n" + text);
        bubble.setTextColor(Color.WHITE);
        bubble.setTextSize(16);
        bubble.setPadding(14, 10, 14, 10);

        LinearLayout.LayoutParams params =
            new LinearLayout.LayoutParams(-1, -2);
        params.setMargins(0, 5, 0, 5);

        bubble.setBackgroundTintList(null);
        bubble.setBackgroundColor(
            who.equals("Kamu")
                ? Color.rgb(35, 70, 120)
                : Color.rgb(35, 43, 62)
        );

        messages.addView(bubble, params);
    }

    private void sendMessage() {
        if (loading) return;

        final String question =
            questionInput.getText().toString().trim();
        final String apiKey =
            keyInput.getText().toString().trim();

        if (apiKey.length() == 0) {
            keyInput.setError("Masukkan API key");
            return;
        }

        if (question.length() == 0) return;

        addMessage("Kamu", question);
        questionInput.setText("");

        InputMethodManager imm =
            (InputMethodManager) getSystemService(
                Context.INPUT_METHOD_SERVICE);
        if (imm != null) {
            imm.hideSoftInputFromWindow(
                questionInput.getWindowToken(), 0);
        }

        try {
            JSONObject userMessage = new JSONObject();
            userMessage.put("role", "user");
            userMessage.put("content", question);
            history.put(userMessage);
        } catch (Exception e) {
            addMessage("AI", "Gagal menyiapkan pesan.");
            return;
        }

        loading = true;
        sendButton.setEnabled(false);
        status.setText("AI sedang berpikir...");

        new Thread(new Runnable() {
            @Override
            public void run() {
                String answer = requestAI(apiKey);

                if (answer == null || answer.length() == 0) {
                    answer = "Maaf, jawaban tidak diterima.";
                }

                final String result = answer;

                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        addMessage("AI", result);

                        if (!result.startsWith("Gagal:") &&
                            !result.startsWith("Error:")) {
                            try {
                                JSONObject assistantMessage =
                                    new JSONObject();
                                assistantMessage.put(
                                    "role", "assistant");
                                assistantMessage.put(
                                    "content", result);
                                history.put(assistantMessage);
                            } catch (Exception ignored) {}
                        } else {
                            // Hapus pesan user terakhir agar
                            // pertanyaan bisa dicoba kembali.
                            history.remove(history.length() - 1);
                        }

                        loading = false;
                        sendButton.setEnabled(true);
                        status.setText("Siap menjawab");
                    }
                });
            }
        }).start();
    }

    private String requestAI(String apiKey) {
        HttpURLConnection connection = null;

        try {
            URL url = new URL(
                "https://api.openai.com/v1/chat/completions");

            connection = (HttpURLConnection)
                url.openConnection();

            connection.setRequestMethod("POST");
            connection.setConnectTimeout(20000);
            connection.setReadTimeout(60000);
            connection.setDoOutput(true);

            connection.setRequestProperty(
                "Content-Type", "application/json");
            connection.setRequestProperty(
                "Authorization", "Bearer " + apiKey);

            JSONObject body = new JSONObject();
            body.put("model", "gpt-4o-mini");
            body.put("messages", history);

            byte[] data = body.toString().getBytes(
                StandardCharsets.UTF_8);

            OutputStream out = connection.getOutputStream();
            out.write(data);
            out.flush();
            out.close();

            int code = connection.getResponseCode();

            InputStream stream = (code >= 200 && code < 300)
                ? connection.getInputStream()
                : connection.getErrorStream();

            String response = readStream(stream);

            JSONObject json = new JSONObject(response);

            if (code < 200 || code >= 300) {
                String message = "Permintaan API gagal.";
                JSONObject error = json.optJSONObject("error");
                if (error != null) {
                    message = error.optString("message", message);
                }
                return "Gagal: HTTP " + code + " - " + message;
            }

            JSONArray choices = json.optJSONArray("choices");

            if (choices == null || choices.length() == 0) {
                return "Gagal: API tidak mengirim jawaban.";
            }

            JSONObject choice = choices.getJSONObject(0);
            JSONObject message = choice.getJSONObject("message");

            return message.optString(
                "content", "Jawaban kosong dari AI.");

        } catch (Exception e) {
            return "Error: " + e.getMessage();
        } finally {
            if (connection != null) {
                connection.disconnect();
            }
        }
    }

    private String readStream(InputStream stream)
        throws IOException {

        if (stream == null) return "";

        ByteArrayOutputStream buffer =
            new ByteArrayOutputStream();

        byte[] bytes = new byte[4096];
        int count;

        try {
            while ((count = stream.read(bytes)) != -1) {
                buffer.write(bytes, 0, count);
            }
            return new String(
                buffer.toByteArray(), StandardCharsets.UTF_8);
        } finally {
            stream.close();
        }
    }
}
