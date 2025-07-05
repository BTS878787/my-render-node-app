const express = require('express');
const axios = require('axios'); // For HTTP requests
const app = express();
app.use(express.json()); // To parse JSON request bodies

const VONAGE_API_KEY = process.env.VONAGE_API_KEY;
const VONAGE_API_SECRET = process.env.VONAGE_API_SECRET;
const WHATSAPP_NUMBER = process.env.WHATSAPP_NUMBER; // Your Vonage WhatsApp number
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;

app.post('/webhook', async (req, res) => {
    console.log('Received webhook:', JSON.stringify(req.body, null, 2));

    // Validate Vonage signature (optional but recommended for security)
    // ...

    const message = req.body.message;
    if (message && message.message_type === 'text') {
        const userText = message.text;
        const fromNumber = message.from; // User's WhatsApp number

        try {
            // 1. Call OpenAI API
            const openaiResponse = await axios.post('https://api.openai.com/v1/chat/completions', {
                model: "gpt-3.5-turbo", // Or "gpt-4"
                messages: [{ role: "user", content: userText }],
                max_tokens: 150
            }, {
                headers: {
                    'Authorization': `Bearer ${OPENAI_API_KEY}`,
                    'Content-Type': 'application/json'
                }
            });

            const chatGptReply = openaiResponse.data.choices[0].message.content;

            // 2. Send reply back via Vonage WhatsApp
            await axios.post('https://api.nexmo.com/v0.1/messages', { // Use https://messages.nexmo.com/v1/messages for new API
                from: { type: 'whatsapp', number: WHATSAPP_NUMBER },
                to: { type: 'whatsapp', number: fromNumber },
                message_type: 'text',
                text: chatGptReply
            }, {
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Basic ' + Buffer.from(`${VONAGE_API_KEY}:${VONAGE_API_SECRET}`).toString('base64')
                }
            });

            res.status(200).send('Message processed');
        } catch (error) {
            console.error('Error:', error.response ? error.response.data : error.message);
            res.status(500).send('Error processing message');
        }
    } else {
        res.status(200).send('No text message received');
    }
});

const PORT = process.env.PORT || 3000; // Render will set process.env.PORT
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
