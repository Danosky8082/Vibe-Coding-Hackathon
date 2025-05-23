# Vibe-Coding-Hackathon

Healthtech: Follow-Up Reminder System

// STEP 1: Supabase Schema (SQL) — paste into Supabase SQL Editor
-- Users Table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  phone TEXT NOT NULL,
  email TEXT,
  role TEXT CHECK (role IN ('doctor', 'patient')) NOT NULL,
  prefers_whatsapp BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT now()
);

-- Appointments Table
CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  doctor_id UUID REFERENCES users(id),
  patient_id UUID REFERENCES users(id),
  datetime TIMESTAMP NOT NULL,
  reminder_sent BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT now()
);

-- Index for performance
CREATE INDEX ON appointments (datetime);


// STEP 2: Backend Logic (Node.js-compatible, e.g., Bolt.new or Lovable.dev)
// Reminder Scheduler Logic (runs every 15 min)
const { createClient } = require('@supabase/supabase-js');
const axios = require('axios');
require('dotenv').config();

const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

async function sendSMS(to, message) {
  return axios.post('https://api.twilio.com/2010-04-01/Accounts/' + process.env.TWILIO_SID + '/Messages.json', new URLSearchParams({
    From: process.env.TWILIO_FROM,
    To: to,
    Body: message
  }), {
    auth: {
      username: process.env.TWILIO_SID,
      password: process.env.TWILIO_AUTH
    }
  });
}

async function remindAppointments() {
  const now = new Date();
  const inAnHour = new Date(now.getTime() + 60 * 60 * 1000);
  const in24Hours = new Date(now.getTime() + 24 * 60 * 60 * 1000);

  const { data: appointments, error } = await supabase
    .from('appointments')
    .select(`*, doctor:doctor_id(*), patient:patient_id(*)`)
    .filter('reminder_sent', 'eq', false)
    .filter('datetime', 'gt', now.toISOString())
    .filter('datetime', 'lte', in24Hours.toISOString());

  if (error) return console.error('Fetch error', error);

  for (const appt of appointments) {
    const message = `📅 Reminder: You have an appointment with Dr. ${appt.doctor.name} on ${new Date(appt.datetime).toLocaleString()}`;
    const to = appt.patient.phone;

    try {
      await sendSMS(to, message);
      await supabase.from('appointments').update({ reminder_sent: true }).eq('id', appt.id);
    } catch (err) {
      console.error('Failed to send SMS', err);
    }
  }
}

remindAppointments();
