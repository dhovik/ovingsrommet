import React, { useMemo, useState, useEffect } from "react";
import { createClient } from '@supabase/supabase-js';

// ============================
// Øvingsrommet • Flow (MVP, Supabase-ready)
// ✅ Nytt i denne versjonen:
//  - Fikset feil: fjernet utilsiktet "$1" i kildekoden som ga ReferenceError
//  - Lagt til <Footer />-komponent (var referert, men ikke definert)
//  - Booking-grupper: "standard", "kulturskole", "kulturenheten" med egne rabatter
//  - Inkluderer group_code + price_nok i bookings (DB og lokal fallback)
//  - Ukes- og månedsvisning: utnyttelsesgrad for valgt uke/måned
//  - Fortsatt: RLS – bare eier kan slette, rom er konstanter, voucher-krav valgfritt, "book for andre" støttes
// ============================

/**
 * --- KJØR DENNE SQL-EN I SUPABASE (SQL Editor) ---
 *
 * -- BOOKINGS (inkl. grupper/pris)
 * create table if not exists public.bookings (
 *   id uuid primary key default gen_random_uuid(),
 *   date date not null,
 *   room_id text not null,
 *   hour int not null check (hour between 0 and 23),
 *   type text not null,
 *   room_name text not null,
 *   voucher_partner text,
 *   booked_for text,
 *   group_code text default 'standard',
 *   price_nok numeric,
 *   created_by uuid not null default auth.uid(),
 *   inserted_at timestamptz default now()
 * );
 * create unique index if not exists bookings_unique_slot on public.bookings(date, room_id, hour);
 * alter table public.bookings enable row level security;
 * create policy if not exists "read_all" on public.bookings for select using (true);
 * create policy if not exists "insert_auth_owns" on public.bookings for insert
 *   with check (auth.role() = 'authenticated' and created_by = auth.uid());
 * create policy if not exists "delete_owner_only" on public.bookings for delete
 *   using (created_by = auth.uid());
 *
 * -- DØR-GRUPPER: map rom -> dører som skal åpnes (ytterdør + romdør)
 * create table if not exists public.door_groups (
 *   room_id text primary key,
 *   door_ids text[] not null
 * );
 * insert into public.door_groups(room_id, door_ids) values
 *   ('s1', array['main','corridor','s1']),
 *   ('s2', array['main','corridor','s2']),
 *   ('b1', array['main','corridor','b1']),
 *   ('b2', array['main','corridor','b2']),
 *   ('b3', array['main','corridor','b3']),
 *   ('b4', array['main','corridor','b4']),
 *   ('b5', array['main','corridor','b5']),
 *   ('p1', array['main','corridor','showroom'])
 * on conflict (room_id) do nothing;
 *
 * -- ACCESS GRANTS: tidsavgrenset nøkkel/PIN per booking
 * create table if not exists public.access_grants (
 *   id uuid primary key default gen_random_uuid(),
 *   booking_id uuid not null references public.bookings(id) on delete cascade,
 *   provider text not null,
 *   door_ids text[] not null,
 *   secret text,      -- PIN-kode (null ved mobilnøkkel)
 *   deep_link text,   -- mobilnøkkel/dyplenke
 *   start_at timestamptz not null,
 *   end_at timestamptz not null,
 *   status text not null default 'issued' check (status in ('issued','revoked','error')),
 *   issued_to text,
 *   created_by uuid not null,
 *   created_at timestamptz default now()
 * );
 * create index if not exists access_grants_booking_idx on public.access_grants(booking_id);
 * alter table public.access_grants enable row level security;
 * create policy if not exists "access_read_owner"
 *   on public.access_grants for select using (
 *     created_by = auth.uid() or exists (
 *       select 1 from public.bookings b where b.id = booking_id and b.created_by = auth.uid()
 *     )
 *   );
 * create policy if not exists "access_delete_owner"
 *   on public.access_grants for delete using (created_by = auth.uid());
 * -- Insert/Update gjøres av Edge Functions (service role)
 *
 * ---------------------------------------------------------------------------
 * SUPABASE EDGE FUNCTIONS (skjelett) – deploy via Supabase CLI
 *   supabase functions new access_get_or_issue
 *   supabase functions new access_revoke
 *   supabase functions deploy access_get_or_issue
 *   supabase functions deploy access_revoke
 * Sett miljøvariabler (Dashboard → Functions):
 *   ACCESS_PROVIDER=pin-demo
 *   DOOR_BUFFER_BEFORE_MIN=15
 *   DOOR_BUFFER_AFTER_MIN=10
 * ---------------------------------------------------------------------------
 * // access_get_or_issue/index.ts (Deno)
 * import { serve } from 'https://deno.land/std@0.181.0/http/server.ts'
 * import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
 * serve(async (req) => {
 *   try {
 *     const { booking_id } = await req.json(); if (!booking_id) return new Response('booking_id missing', { status: 400 });
 *     const supabase = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);
 *     const { data: booking } = await supabase.from('bookings').select('id,date,hour,room_id,created_by,booked_for').eq('id', booking_id).single();
 *     const nowISO = new Date().toISOString();
 *     const { data: existing } = await supabase.from('access_grants').select('*').eq('booking_id', booking_id).eq('status','issued').gte('end_at', nowISO).limit(1);
 *     if (existing && existing.length) return Response.json(existing[0]);
 *     const { data: dg } = await supabase.from('door_groups').select('door_ids').eq('room_id', booking.room_id).maybeSingle();
 *     const door_ids = dg?.door_ids || ['main'];
 *     const before = Number(Deno.env.get('DOOR_BUFFER_BEFORE_MIN')||'15');
 *     const after = Number(Deno.env.get('DOOR_BUFFER_AFTER_MIN')||'10');
 *     const start = new Date(`${booking.date}T${String(booking.hour).padStart(2,'0')}:00:00Z`);
 *     start.setMinutes(start.getMinutes()-before);
 *     const end = new Date(`${booking.date}T${String(booking.hour+1).padStart(2,'0')}:00:00Z`);
 *     end.setMinutes(end.getMinutes()+after);
 *     const pin = String(Math.floor(100000 + Math.random()*900000));
 *     const payload = { booking_id, provider: Deno.env.get('ACCESS_PROVIDER')||'pin-demo', door_ids, secret: pin, deep_link: null, start_at: start.toISOString(), end_at: end.toISOString(), status: 'issued', issued_to: booking.booked_for||null, created_by: booking.created_by };
 *     const { data: grant, error: ge } = await supabase.from('access_grants').insert(payload).select('*').single();
 *     if (ge) return new Response(ge.message, { status: 500 });
 *     return Response.json(grant);
 *   } catch (e) { return new Response(String(e?.message||e), { status: 500 }); }
 * });
 *
 * // access_revoke/index.ts (Deno)
 * import { serve } from 'https://deno.land/std@0.181.0/http/server.ts'
 * import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
 * serve(async (req) => {
 *   try {
 *     const { booking_id } = await req.json(); if (!booking_id) return new Response('booking_id missing', { status: 400 });
 *     const supabase = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);
 *     const { error } = await supabase.from('access_grants').update({ status: 'revoked' }).eq('booking_id', booking_id).eq('status','issued');
 *     if (error) return new Response(error.message, { status: 500 });
 *     return Response.json({ ok: true });
 *   } catch (e) { return new Response(String(e?.message||e), { status: 500 }); }
 * });
 */

// ----- Config -----
const OPEN_HOUR = 10; // 10:00
const CLOSE_HOUR = 23; // 23:00 (exclusive end)
const HOURS_PER_DAY = CLOSE_HOUR - OPEN_HOUR; // 13
const BREAK_EVEN = 17.4; // % utilization target

const DEFAULT_ROOMS = [
  { id: "s1", name: "Solo 1", type: "solo" },
  { id: "s2", name: "Solo 2", type: "solo" },
  { id: "b1", name: "Band 1", type: "band" },
  { id: "b2", name: "Band 2", type: "band" },
  { id: "b3", name: "Band 3", type: "band" },
  { id: "b4", name: "Band 4", type: "band" },
  { id: "b5", name: "Band 5", type: "band" },
  { id: "p1", name: "Preprod / Scene", type: "preprod" },
];

// Standard satser (kan justeres i Admin senere om ønskelig)
const RATECARD = { solo: 199, band: 399, preprod: 799 };

// Booking-grupper (multiplikatorer). Kan endres i Admin.
const DEFAULT_PRICING = {
  base: RATECARD,
  groups: {
    standard: 1.0,
    kulturskole: 0.7,     // 30% rabatt som utgangspunkt
    kulturenheten: 0.75,  // 25% rabatt som utgangspunkt
  }
};

// Energi (enkle demo-tall – juster i Admin)
const DEFAULT_ENERGY = { solo: 3.0, band: 4.5, preprod: 7.0, optimizationFactor: 0.88 };

// ----- ENV / Supabase client -----
const SUPABASE_URL = import.meta?.env?.VITE_SUPABASE_URL;
const SUPABASE_ANON_KEY = import.meta?.env?.VITE_SUPABASE_ANON_KEY;
const hasSupabase = Boolean(SUPABASE_URL && SUPABASE_ANON_KEY);
const supabase = hasSupabase ? createClient(SUPABASE_URL, SUPABASE_ANON_KEY) : null;

// ----- Helpers -----
const fmtDate = (d) => new Date(d.getTime() - d.getTimezoneOffset()*60000).toISOString().slice(0,10);
const todayISO = () => fmtDate(new Date());
const nextId = () => Math.random().toString(36).slice(2,8);
const clone = (obj) => { try { return structuredClone(obj); } catch { return JSON.parse(JSON.stringify(obj)); } };
function hoursArray() { return Array.from({ length: HOURS_PER_DAY }, (_, i) => OPEN_HOUR + i); }
function generateAccessCode(booking) { const seed = `${booking.date}|${booking.roomId}|${booking.hour}`; let h=0; for (let i=0;i<seed.length;i++) h=(h*31+seed.charCodeAt(i))>>>0; return (h%1000000).toString().padStart(6,"0"); }
function saveLS(key, value) { localStorage.setItem(key, JSON.stringify(value)); }
function loadLS(key, fallback) { try { const v = localStorage.getItem(key); return v ? JSON.parse(v) : fallback; } catch { return fallback; } }

// Dates
function parseISO(s){ const [y,m,d]=s.split('-').map(Number); const dt = new Date(Date.UTC(y,m-1,d)); return dt; }
function addDaysISO(s, days){ const dt=parseISO(s); dt.setUTCDate(dt.getUTCDate()+days); return fmtDate(dt); }
function startOfWeekISO(s){ // mandag som første dag
  const dt=parseISO(s); const day = dt.getUTCDay(); // 0=Sun..6=Sat
  const diff = (day===0? -6 : 1 - day); // til mandag
  return addDaysISO(s, diff);
}
function endOfWeekISO(s){ const start = startOfWeekISO(s); return addDaysISO(start, 6); }
function startOfMonthISO(s){ const dt=parseISO(s); dt.setUTCDate(1); return fmtDate(dt); }
function endOfMonthISO(s){ const dt=parseISO(s); dt.setUTCMonth(dt.getUTCMonth()+1,0); return fmtDate(dt); }
function eachDateISO(startISO, endISO){ const out=[]; let d=startISO; while(d<=endISO){ out.push(d); d = addDaysISO(d,1); } return out; }

// Voucher utils (pure)
function checkVoucherAvailable(vouchers, id){ const v = vouchers.find(x=>x.id===id); return !!(v && v.slots>0); }
function adjustVoucherSlots(vouchers, id, delta){ return vouchers.map(v => v.id===id ? { ...v, slots: Math.max(0, (v.slots||0)+delta) } : v); }
function findVoucherByPartner(vouchers, partner){ return vouchers.find(v=>v.partner===partner); }

// Booking mode helper (pure)
function determineBookingMode({ voucherRequired, bookForOthers }){
  if (bookForOthers) return 'external';
  if (voucherRequired) return 'voucher';
  return 'open';
}

// Pricing helpers
function computePrice(roomType, pricing, groupCode){
  const base = pricing?.base?.[roomType] ?? RATECARD[roomType] ?? 0;
  const mult = pricing?.groups?.[groupCode||'standard'] ?? 1.0;
  return Math.round(base * mult);
}

// ----- App Root -----
export default function App() {
  // Brand siden i nettleseren
  useEffect(() => { document.title = "Øvingsrommet"; }, []);
  const [rooms, setRooms] = useState(loadLS("rooms", DEFAULT_ROOMS));
  const [dateISO, setDateISO] = useState(todayISO());
  const [bookings, setBookings] = useState(loadLS("bookings", {})); // fallback only
  const [vouchers, setVouchers] = useState(loadLS("vouchers", [
    { id: nextId(), partner: "Ung Kultur Lerkendal", slots: 40 },
    { id: nextId(), partner: "Fritidsklubb Midtbyen", slots: 30 },
  ]));
  const [energy, setEnergy] = useState(loadLS("energy", DEFAULT_ENERGY));
  const [pricing, setPricing] = useState(loadLS("pricing", DEFAULT_PRICING));
  const [view, setView] = useState("dashboard");
  const [busyCells, setBusyCells] = useState(new Set());
  const [notice, setNotice] = useState("");
  const [voucherRequired, setVoucherRequired] = useState(true);
  const [activeVoucherId, setActiveVoucherId] = useState(vouchers[0]?.id || "");
  const [bookForOthers, setBookForOthers] = useState(false);
  const [bookedFor, setBookedFor] = useState(""); // navn/epost
  const [activeGroup, setActiveGroup] = useState('standard');
  const [accessModal, setAccessModal] = useState({ open:false, grant:null, booking:null, error:null, loading:false });

  // Range bookings for week/month (Supabase) – maps by dateISO -> byRoom
  const [rangeWeek, setRangeWeek] = useState({});
  const [rangeMonth, setRangeMonth] = useState({});

  // Auth
  const [session, setSession] = useState(null);
  useEffect(() => { if(!hasSupabase) return; (async()=>{ const { data } = await supabase.auth.getSession(); setSession(data.session||null); supabase.auth.onAuthStateChange((_evt, s)=> setSession(s)); })(); }, []);

  // Persist local parts
  useEffect(() => saveLS("rooms", rooms), [rooms]);
  useEffect(() => saveLS("vouchers", vouchers), [vouchers]);
  useEffect(() => saveLS("energy", energy), [energy]);
  useEffect(() => saveLS("pricing", pricing), [pricing]);

  // Self-tests
  useEffect(() => { runSelfTests(); }, []);

  // Load bookings for date (Supabase > fallback)
  useEffect(() => { (async () => { if (hasSupabase) await refreshBookings(dateISO, setBookings); })(); }, [dateISO]);

  // Load week & month ranges when date changes (Supabase) – else use local store
  useEffect(() => { (async () => {
    const ws = startOfWeekISO(dateISO), we = endOfWeekISO(dateISO);
    const ms = startOfMonthISO(dateISO), me = endOfMonthISO(dateISO);
    if (hasSupabase) {
      const week = await refreshRange(ws, we);
      const month = await refreshRange(ms, me);
      setRangeWeek(week); setRangeMonth(month);
    } else {
      // local: filter existing local bookings map
      setRangeWeek(filterRangeLocal(bookings, ws, we));
      setRangeMonth(filterRangeLocal(bookings, ms, me));
    }
  })(); }, [dateISO, hasSupabase]);

  const stats = useMemo(() => computeStats({ bookings, dateISO, rooms, energy }), [bookings, dateISO, rooms, energy]);

  const weekStats = useMemo(() => computeUtilizationRange(rangeWeek, rooms, startOfWeekISO(dateISO), endOfWeekISO(dateISO)), [rangeWeek, rooms, dateISO]);
  const monthStats = useMemo(() => computeUtilizationRange(rangeMonth, rooms, startOfMonthISO(dateISO), endOfMonthISO(dateISO)), [rangeMonth, rooms, dateISO]);

  // ----- Booking handlers -----
  const handleCreate = async (b) => {
    const mode = determineBookingMode({ voucherRequired, bookForOthers });

    // Voucher gate
    if (mode === 'voucher') {
      if (!activeVoucherId) return setNotice('Velg et aktivt klippekort før booking.');
      if (!checkVoucherAvailable(vouchers, activeVoucherId)) return setNotice('Ingen klipp igjen på valgt klippekort.');
    }

    const groupCode = activeGroup || 'standard';
    const price = computePrice(b.type, pricing, groupCode);

    if (hasSupabase) {
      if (!session) return setNotice('Du må være innlogget for å booke.');
      const key = `${b.roomId}-${b.hour}`; setBusyCells(new Set(busyCells).add(key));
      try {
        const voucherPartner = mode === 'voucher' ? (vouchers.find(v=>v.id===activeVoucherId)?.partner || null) : null;
        const bookedForVal = mode === 'external' ? (bookedFor?.trim() || null) : null;
        const { error } = await supabase.from('bookings').insert({
          date: b.date, room_id: b.roomId, hour: b.hour, type: b.type, room_name: b.roomName,
          voucher_partner: voucherPartner, booked_for: bookedForVal, group_code: groupCode, price_nok: price,
        }).select('id').single();
        if (error) {
          if (error.code === '23505') setNotice('Tidslottet er allerede booket.'); else setNotice(`Feil ved booking: ${error.message}`);
        } else {
          if (mode === 'voucher' && activeVoucherId) setVouchers(prev => adjustVoucherSlots(prev, activeVoucherId, -1));
          await refreshBookings(b.date, setBookings);
          if (mode === 'external') setBookedFor("");
          // refresh ranges for fresh stats
          const ws = startOfWeekISO(dateISO), we = endOfWeekISO(dateISO);
          const ms = startOfMonthISO(dateISO), me = endOfMonthISO(dateISO);
          if (hasSupabase) {
            setRangeWeek(await refreshRange(ws, we));
            setRangeMonth(await refreshRange(ms, me));
          }
        }
      } finally { const s = new Set(busyCells); s.delete(key); setBusyCells(s); }
    } else {
      // Local fallback
      const modeVoucherPartner = mode === 'voucher' ? (vouchers.find(v=>v.id===activeVoucherId)?.partner || null) : null;
      const bookedForVal = mode === 'external' ? (bookedFor?.trim() || null) : null;
      const book = { ...b, voucherPartner: modeVoucherPartner, bookedFor: bookedForVal, groupCode, priceNOK: price, createdBy: 'local' };
      const newStore = addBooking(bookings, book);
      setBookings(newStore);
      saveLS("bookings", newStore);
      if (mode === 'voucher' && activeVoucherId) setVouchers(prev => adjustVoucherSlots(prev, activeVoucherId, -1));
      if (mode === 'external') setBookedFor("");
      // update local ranges
      setRangeWeek(filterRangeLocal(newStore, startOfWeekISO(dateISO), endOfWeekISO(dateISO)));
      setRangeMonth(filterRangeLocal(newStore, startOfMonthISO(dateISO), endOfMonthISO(dateISO)));
    }
  };

  const handleDelete = async (b) => {
    if (hasSupabase) {
      if (!session) return setNotice('Du må være innlogget for å slette.');
      const key = `${b.roomId}-${b.hour}`; setBusyCells(new Set(busyCells).add(key));
      try {
        // Finn booking for å vite hvilken voucher som skal refunderes
        const existing = await supabase.from('bookings')
          .select('id, voucher_partner, created_by')
          .eq('date', b.date).eq('room_id', b.roomId).eq('hour', b.hour).maybeSingle();
        const bookingId = existing.data?.id;
        const voucherPartner = existing.data?.voucher_partner || null;
        if (bookingId) {
          try { await supabase.functions.invoke('access_revoke', { body: { booking_id: bookingId } }); } catch (_) {}
        }
        const { error } = await supabase.from('bookings')
          .delete()
          .eq('date', b.date).eq('room_id', b.roomId).eq('hour', b.hour);
        if (error) {
          setNotice('Kun eier av bookingen kan slette denne.');
        } else {
          if (voucherPartner) {
            const v = findVoucherByPartner(vouchers, voucherPartner);
            if (v) setVouchers(prev => adjustVoucherSlots(prev, v.id, +1));
          }
          await refreshBookings(b.date, setBookings);
          // refresh ranges
          const ws = startOfWeekISO(dateISO), we = endOfWeekISO(dateISO);
          const ms = startOfMonthISO(dateISO), me = endOfMonthISO(dateISO);
          setRangeWeek(await refreshRange(ws, we));
          setRangeMonth(await refreshRange(ms, me));
        }
      } finally { const s = new Set(busyCells); s.delete(key); setBusyCells(s); }
    } else {
      // Local fallback
      const cell = bookings[b.date]?.[b.roomId]?.[b.hour];
      if (cell?.voucherPartner) {
        const v = findVoucherByPartner(vouchers, cell.voucherPartner);
        if (v) setVouchers(prev => adjustVoucherSlots(prev, v.id, +1));
      }
      const newStore = removeBooking(bookings, b);
      setBookings(newStore);
      saveLS("bookings", newStore);
      setRangeWeek(filterRangeLocal(newStore, startOfWeekISO(dateISO), endOfWeekISO(dateISO)));
      setRangeMonth(filterRangeLocal(newStore, startOfMonthISO(dateISO), endOfMonthISO(dateISO)));
    }
  };

  const showAccessFor = async (b) => {
    if (!b) return;
    if (!hasSupabase) {
      const before = 15, after = 10;
      const start = new Date(`${b.date}T${String(b.hour).padStart(2,'0')}:00:00Z`);
      start.setMinutes(start.getMinutes()-before);
      const end = new Date(`${b.date}T${String(b.hour+1).padStart(2,'0')}:00:00Z`);
      end.setMinutes(end.getMinutes()+after);
      setAccessModal({ open:true, booking:b, grant:{ provider:'demo-pin', secret: generateAccessCode(b), start_at: start.toISOString(), end_at: end.toISOString() }, error:null, loading:false });
      return;
    }
    if (!session) { setNotice('Du må være innlogget for å vise tilgang.'); return; }
    setAccessModal({ open:true, booking:b, grant:null, error:null, loading:true });
    const { data, error } = await supabase.functions.invoke('access_get_or_issue', { body: { booking_id: b.id } });
    if (error) setAccessModal({ open:true, booking:b, grant:null, error: error.message, loading:false });
    else setAccessModal({ open:true, booking:b, grant:data, error:null, loading:false });
  };

  return (
    <div className="min-h-screen bg-neutral-50 text-neutral-900 flex flex-col">
      <Header
        view={view} setView={setView}
        stats={stats} dateISO={dateISO} setDateISO={setDateISO}
        hasSupabase={hasSupabase}
        session={session}
        vouchers={vouchers}
        voucherRequired={voucherRequired}
        setVoucherRequired={setVoucherRequired}
        activeVoucherId={activeVoucherId}
        setActiveVoucherId={setActiveVoucherId}
        bookForOthers={bookForOthers}
        setBookForOthers={setBookForOthers}
        bookedFor={bookedFor}
        setBookedFor={setBookedFor}
        pricing={pricing}
        setPricing={setPricing}
        activeGroup={activeGroup}
        setActiveGroup={setActiveGroup}
      />
      {notice && (
        <div className="mx-auto max-w-7xl px-4 mt-3">
          <div className="bg-yellow-50 border border-yellow-200 text-yellow-800 text-sm px-3 py-2 rounded">{notice}</div>
        </div>
      )}
      <main className="mx-auto max-w-7xl px-4 pb-24 w-full grow">
        {view === "dashboard" && <Dashboard stats={stats} weekStats={weekStats} monthStats={monthStats} hasSupabase={hasSupabase} session={session} onShowAccess={showAccessFor} />}
        {view === "book" && (
          <BookingView
            rooms={rooms}
            bookings={bookings}
            dateISO={dateISO}
            onCreate={handleCreate}
            onDelete={handleDelete}
            busyCells={busyCells}
            canDelete={(cell)=> {
              if (!cell) return false;
              if (!hasSupabase) return true; // local fallback
              if (!session) return false;
              return cell.createdBy && cell.createdBy === session.user.id;
            }}
          />
        )}
        {view === "vouchers" && (
          <VoucherView vouchers={vouchers} setVouchers={setVouchers} />
        )}
        {view === "energy" && (
          <EnergyView stats={stats} energy={energy} setEnergy={setEnergy} />
        )}
        {view === "admin" && (
          <AdminView rooms={rooms} setRooms={setRooms} bookings={bookings} dateISO={dateISO} pricing={pricing} setPricing={setPricing} />
        )}
      </main>
      <Footer />
      <AccessModal open={accessModal.open} onClose={()=>setAccessModal({...accessModal, open:false})} grant={accessModal.grant} booking={accessModal.booking} error={accessModal.error} loading={accessModal.loading} />
    </div>
  );
}

// ----- Header / Nav + Auth -----
function Header({ view, setView, stats, dateISO, setDateISO, hasSupabase, session, vouchers, voucherRequired, setVoucherRequired, activeVoucherId, setActiveVoucherId, bookForOthers, setBookForOthers, bookedFor, setBookedFor, pricing, setPricing, activeGroup, setActiveGroup }) {
  const tabs = [
    { id: "dashboard", label: "Dashboard" },
    { id: "book", label: "Booking" },
    { id: "vouchers", label: "Vouchers" },
    { id: "energy", label: "Energi" },
    { id: "admin", label: "Admin" },
  ];
  const voucherUIEnabled = !bookForOthers; // når man booker for andre, skal klippekort ikke kreves/brukes
  return (
    <header className="sticky top-0 z-10 backdrop-blur bg-white/80 border-b border-neutral-200">
      <div className="mx-auto max-w-7xl px-4 py-3 flex items-center gap-4">
        <div className="font-bold text-lg">Øvingsrommet</div>
        <nav className="flex gap-1 text-sm">
          {tabs.map(t => (
            <button key={t.id}
              onClick={() => setView(t.id)}
              className={`px-3 py-1.5 rounded-full border ${view===t.id?"bg-neutral-900 text-white border-neutral-900":"bg-white hover:bg-neutral-100 border-neutral-200"}`}>{t.label}</button>
          ))}
        </nav>
        <div className="ml-auto flex items-center gap-3">
          <input type="date" value={dateISO} onChange={e=>setDateISO(e.target.value)} className="px-3 py-1.5 rounded-md border border-neutral-300 text-sm"/>
          <div className="hidden md:flex gap-3">
            <StatPill label="Utnyttelse i dag" value={`${stats.utilization.toFixed(1)}%`} ok={stats.utilization>=BREAK_EVEN} />
            <StatPill label="Inntekt (est)" value={`${formatNOK(stats.revenueToday)}`} />
            <StatPill label="kWh/time (baseline)" value={stats.kwhPerBookedHour.toFixed(2)} />
          </div>
          <span className={`text-xs px-2 py-1 rounded border ${hasSupabase?"bg-green-50 text-green-700 border-green-200":"bg-neutral-50 text-neutral-600 border-neutral-200"}`}>
            {hasSupabase? 'DB: Supabase' : 'DB: Lokal demo'}
          </span>
        </div>
      </div>
      <div className="mx-auto max-w-7xl px-4 pb-3 flex flex-wrap items-center gap-3 text-sm">
        <div className="flex items-center gap-2">
          <input id="bookForOthers" type="checkbox" checked={bookForOthers} onChange={e=>setBookForOthers(e.target.checked)} />
          <label htmlFor="bookForOthers">Book for andre (uten klippekort)</label>
        </div>
        {bookForOthers && (
          <div className="flex items-center gap-2">
            <span>Navn/epost:</span>
            <input value={bookedFor} onChange={e=>setBookedFor(e.target.value)} placeholder="Navn eller epost" className="px-2 py-1 border rounded min-w-[220px]" />
          </div>
        )}
        <div className="flex items-center gap-2">
          <span>Gruppe:</span>
          <select value={activeGroup} onChange={e=>setActiveGroup(e.target.value)} className="px-2 py-1 border rounded">
            <option value="standard">Standard</option>
            <option value="kulturskole">Kulturskole</option>
            <option value="kulturenheten">Kulturenheten</option>
          </select>
          <span className="text-neutral-500">(solo {formatNOK(computePrice('solo',pricing,activeGroup))}, band {formatNOK(computePrice('band',pricing,activeGroup))}, preprod {formatNOK(computePrice('preprod',pricing,activeGroup))})</span>
        </div>
        <div className={`flex items-center gap-2 ${voucherUIEnabled? '' : 'opacity-50 pointer-events-none'}`}>
          <input id="voucherlock" type="checkbox" checked={voucherRequired} onChange={e=>setVoucherRequired(e.target.checked)} />
          <label htmlFor="voucherlock">Krev klippekort ved booking</label>
        </div>
        <div className={`flex items-center gap-2 ${voucherUIEnabled? '' : 'opacity-50 pointer-events-none'}`}>
          <span>Aktivt klippekort:</span>
          <select value={activeVoucherId} onChange={e=>setActiveVoucherId(e.target.value)} className="px-2 py-1 border rounded">
            {vouchers.map(v => <option key={v.id} value={v.id}>{v.partner} ({v.slots})</option>)}
          </select>
        </div>
        {hasSupabase && (
          <div className="ml-auto">
            {session ? <AuthBadge session={session} /> : <AuthPanel />}
          </div>
        )}
      </div>
    </header>
  );
}

function AuthBadge({ session }){
  const email = session?.user?.email || 'Innlogget';
  return (
    <div className="flex items-center gap-2">
      <span className="text-neutral-600">{email}</span>
      <button className="px-2 py-1 rounded border" onClick={()=>supabase.auth.signOut()}>Logg ut</button>
    </div>
  );
}

function AuthPanel(){
  const [email, setEmail] = useState("");
  const [sent, setSent] = useState(false);
  const send = async ()=>{
    if (!email) return;
    const { error } = await supabase.auth.signInWithOtp({ email, options: { emailRedirectTo: window.location.origin } });
    if (error) alert(error.message); else setSent(true);
  };
  return (
    <div className="flex items-center gap-2">
      {sent ? <span className="text-neutral-600">Sjekk e-posten din for innloggingslenke</span> : (
        <>
          <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="din@epost.no" className="px-2 py-1 border rounded"/>
          <button className="px-2 py-1 rounded border" onClick={send}>Logg inn</button>
        </>
      )}
    </div>
  );
}

function StatPill({ label, value, ok }) {
  return (
    <div className={`px-3 py-1.5 rounded-full text-sm border ${ok?"bg-green-50 text-green-800 border-green-200":"bg-neutral-50 text-neutral-700 border-neutral-200"}`}>
      <span className="mr-2 opacity-70">{label}</span>
      <span className="font-semibold">{value}</span>
    </div>
  );
}

// ----- Dashboard -----
function Dashboard({ stats, weekStats, monthStats, hasSupabase, session, onShowAccess }) {
  return (
    <section className="mt-6 grid md:grid-cols-3 gap-4">
      <Card title="Utnyttelse i dag">
        <div className="flex items-end gap-4">
          <BigNumber value={`${stats.utilization.toFixed(1)}%`} subt="Break-even 17,4%" ok={stats.utilization>=BREAK_EVEN} />
          <Bars percentage={stats.utilization} />
        </div>
      </Card>
      <Card title="Estimert inntekt i dag">
        <BigNumber value={formatNOK(stats.revenueToday)} subt="Sum av faktiske slot-priser" />
      </Card>
      <Card title="Energi (kWh per brukstime)">
        <BigNumber value={stats.kwhPerBookedHour.toFixed(2)} subt={`Baseline i dag`} />
      </Card>

      <Card title="Utnyttelse denne uken (man–søn)">
        <div className="flex items-end gap-4">
          <BigNumber value={`${weekStats.utilization.toFixed(1)}%`} subt={`${weekStats.booked}/${weekStats.total} timer`} />
          <Bars percentage={weekStats.utilization} />
        </div>
      </Card>
      <Card title="Utnyttelse denne måneden">
        <div className="flex items-end gap-4">
          <BigNumber value={`${monthStats.utilization.toFixed(1)}%`} subt={`${monthStats.booked}/${monthStats.total} timer`} />
          <Bars percentage={monthStats.utilization} />
        </div>
      </Card>

      <Card title="Dagens bookinger">
        {stats.todayList.length===0 && <div className="text-sm text-neutral-500">Ingen bookinger ennå. Gå til Booking for å legge inn.</div>}
        <ul className="text-sm divide-y">
          {stats.todayList.map((b)=> {
            const canAccess = hasSupabase ? (session && b.createdBy === session.user.id) : true;
            return (
              <li key={`${b.roomId}-${b.hour}`} className="py-2 flex items-center justify-between">
                <span>
                  <span className="font-medium mr-2">{b.roomName}</span>
                  <span className="text-neutral-500">{fmtHour(b.hour)}–{fmtHour(b.hour+1)} • {b.typeLabel}</span>
                  {b.voucherPartner && <span className="ml-2 text-xs text-neutral-500">• via {b.voucherPartner}</span>}
                  {b.bookedFor && <span className="ml-2 text-xs text-neutral-500">• for {b.bookedFor}</span>}
                  {b.groupCode && b.groupCode!=='standard' && <span className="ml-2 text-xs text-neutral-500">• {b.groupCode}</span>}
                </span>
                <div className="flex items-center gap-2">
                  {typeof b.priceNOK === 'number' && <span className="text-xs text-neutral-600">{formatNOK(b.priceNOK)}</span>}
                  {!hasSupabase && <code className="text-xs bg-neutral-100 px-2 py-1 rounded">kode {generateAccessCode(b)}</code>}
                  {canAccess && <button className="px-2 py-1 text-xs rounded border" onClick={()=>onShowAccess(b)}>Tilgang</button>}
                </div>
              </li>
            );
          })}
        </ul>
      </Card>
      <Card title="Rask eksport (CSV)">
        <p className="text-sm text-neutral-600 mb-2">Eksporter bookinger for valgt dato.</p>
        <button className="px-3 py-2 rounded-md bg-neutral-900 text-white text-sm" onClick={()=>exportCSV(stats.todayList)}>Last ned CSV</button>
      </Card>
      <Card title="Break-even indikator">
        <p className="text-sm">Utnyttelse i dag: <b>{stats.utilization.toFixed(1)}%</b>. Break-even: <b>{BREAK_EVEN}%</b>.</p>
        <div className="h-2 bg-neutral-200 rounded mt-3 overflow-hidden">
          <div className="h-full bg-neutral-900" style={{ width: `${Math.min(100, stats.utilization)}%` }} />
        </div>
      </Card>
    </section>
  );
}

function Bars({ percentage }) {
  const bars = Array.from({length: 10}, (_,i)=> i*10 < percentage);
  return (
    <div className="flex gap-1 items-end ml-auto">
      {bars.map((on,i)=> <div key={i} className={`w-2 rounded ${on?"bg-neutral-900":"bg-neutral-200"}`} style={{height: `${6+i*6}px`}} />)}
    </div>
  );
}

function BigNumber({ value, subt, ok }) {
  return (
    <div>
      <div className={`text-3xl font-semibold ${ok?"text-green-700":"text-neutral-900"}`}>{value}</div>
      <div className="text-sm text-neutral-500">{subt}</div>
    </div>
  );
}

function Card({ title, children }) {
  return (
    <div className="bg-white rounded-2xl border border-neutral-200 p-4 shadow-sm">
      {title && <div className="font-semibold mb-2">{title}</div>}
      {children}
    </div>
  );
}

// ----- Booking View -----
function BookingView({ rooms, bookings, dateISO, onCreate, onDelete, busyCells, canDelete }) {
  const hours = hoursArray();
  const dayBookings = bookings[dateISO] || {};

  const makeBooking = (roomId, hour) => {
    const room = rooms.find(r=>r.id===roomId);
    const b = { id: nextId(), date: dateISO, roomId, hour, type: room.type, typeLabel: roomTypeLabel(room.type), roomName: room.name };
    onCreate(b);
  };

  const remove = (roomId, hour) => { onDelete({ date: dateISO, roomId, hour }); };

  return (
    <section className="mt-6">
      <div className="mb-3 flex items-center justify-between">
        <h2 className="font-semibold">Booking – {dateISO}</h2>
        <div className="text-sm text-neutral-600">Åpent {OPEN_HOUR}:00–{CLOSE_HOUR}:00 • {rooms.length} rom</div>
      </div>
      <div className="overflow-x-auto">
        <table className="w-full text-sm border-collapse">
          <thead>
            <tr>
              <th className="text-left p-2 sticky left-0 bg-neutral-50">Rom</th>
              {hours.map(h=> <th key={h} className="p-2 text-center border-b border-neutral-200 min-w-[60px]">{fmtHour(h)}</th>)}
            </tr>
          </thead>
          <tbody>
            {rooms.map(room => (
              <tr key={room.id} className="odd:bg-white even:bg-neutral-50">
                <td className="p-2 font-medium sticky left-0 bg-inherit">{room.name} <span className="text-xs text-neutral-500">• {roomTypeLabel(room.type)}</span></td>
                {hours.map(h => {
                  const cell = dayBookings[room.id]?.[h];
                  const taken = !!cell;
                  const cellKey = `${room.id}-${h}`;
                  const isBusy = busyCells?.has(cellKey);
                  const deletable = taken && canDelete(cell);
                  return (
                    <td key={h} className="p-1 text-center border-b border-neutral-100">
                      {taken ? (
                        <button disabled={!deletable || isBusy} className={`w-full py-2 rounded text-white ${(!deletable||isBusy)? 'bg-neutral-400' : 'bg-neutral-900 hover:bg-neutral-800'}`} onClick={()=>remove(room.id, h)}>
                          {isBusy ? '…' : deletable ? `Slett ${fmtHour(h)}–${fmtHour(h+1)}` : 'Booket'}
                        </button>
                      ) : (
                        <button disabled={isBusy} className={`w-full py-2 rounded border ${isBusy? 'bg-neutral-100 text-neutral-400 border-neutral-200' : 'bg-white border-neutral-300 hover:bg-neutral-100'}`} onClick={()=>makeBooking(room.id, h)}>
                          {isBusy ? '…' : 'Ledig'}
                        </button>
                      )}
                    </td>
                  );
                })}
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      <p className="text-xs text-neutral-500 mt-2">Tips: Klikk «Ledig» for å opprette booking. Du kan kun slette egne bookinger.</p>
    </section>
  );
}

// ----- Vouchers -----
function VoucherView({ vouchers, setVouchers }) {
  const [partner, setPartner] = useState("");
  const [slots, setSlots] = useState(10);
  return (
    <section className="mt-6 grid md:grid-cols-2 gap-4">
      <Card title="Aktive vouchers">
        <ul className="divide-y">
          {vouchers.map(v => (
            <li key={v.id} className="py-2 flex items-center justify-between">
              <div>
                <div className="font-medium">{v.partner}</div>
                <div className="text-xs text-neutral-500">Klippekort: {v.slots} slots igjen</div>
              </div>
              <div className="flex items-center gap-2">
                <button className="px-2 py-1 text-sm rounded border" onClick={()=> setVouchers(adjustVoucherSlots(vouchers, v.id, +5))}>+5</button>
                <button className="px-2 py-1 text-sm rounded border" onClick={()=> setVouchers(adjustVoucherSlots(vouchers, v.id, -5))}>-5</button>
                <button className="px-2 py-1 text-sm rounded border" onClick={()=> setVouchers(vouchers.filter(x => x.id!==v.id))}>Slett</button>
              </div>
            </li>
          ))}
        </ul>
      </Card>
      <Card title="Opprett nytt klippekort">
        <div className="flex flex-col gap-2">
          <label className="text-sm">Partner/klubb</label>
          <input value={partner} onChange={e=>setPartner(e.target.value)} placeholder="Fritidsklubb / skole / organisasjon" className="px-3 py-2 rounded border" />
          <label className="text-sm">Antall slots</label>
          <input type="number" value={slots} onChange={e=>setSlots(Number(e.target.value))} className="px-3 py-2 rounded border w-32" />
          <button className="px-3 py-2 rounded-md bg-neutral-900 text-white w-fit" onClick={()=>{
            if(!partner) return;
            setVouchers([...vouchers, { id: nextId(), partner, slots: Number(slots)||0 }]); setPartner(""); setSlots(10);
          }}>Opprett</button>
        </div>
        <p className="text-xs text-neutral-500 mt-3">Aktiver krav i toppbaren for å kreve klipp ved booking.</p>
      </Card>
    </section>
  );
}

// ----- Energy -----
function EnergyView({ stats, energy, setEnergy }) {
  const baseline = useMemo(()=>computeBaselineEnergy(stats), [stats]);
  const optimized = baseline * energy.optimizationFactor;
  return (
    <section className="mt-6 grid md:grid-cols-2 gap-4">
      <Card title="kWh per brukstime (estimat)">
        <div className="flex items-end gap-6">
          <div>
            <div className="text-sm text-neutral-500">Baseline</div>
            <div className="text-2xl font-semibold">{baseline.toFixed(2)} kWh/t</div>
          </div>
          <div>
            <div className="text-sm text-neutral-500">Mål etter tiltak</div>
            <div className="text-2xl font-semibold text-green-700">{optimized.toFixed(2)} kWh/t</div>
          </div>
          <div className="ml-auto">
            <div className="text-sm text-neutral-500">Reduksjon</div>
            <div className="text-2xl font-semibold">{((1-energy.optimizationFactor)*100).toFixed(0)}%</div>
          </div>
        </div>
        <div className="mt-4 h-3 bg-neutral-200 rounded overflow-hidden">
          <div className="h-full bg-neutral-900" style={{ width: `${Math.min(100, (optimized/baseline)*100)}%` }} />
        </div>
        <p className="text-xs text-neutral-500 mt-2">Forenklet beregning basert på fordeling av bookede timer per romtype.</p>
      </Card>
      <Card title="Parametre (juster og se effekt)">
        <div className="grid grid-cols-2 gap-3 text-sm">
          <LabeledInput label="Solo kWh/h" value={energy.solo} onChange={v=> setEnergy({...energy, solo:Number(v)||0})} />
          <LabeledInput label="Band kWh/h" value={energy.band} onChange={v=> setEnergy({...energy, band:Number(v)||0})} />
          <LabeledInput label="Preprod kWh/h" value={energy.preprod} onChange={v=> setEnergy({...energy, preprod:Number(v)||0})} />
          <LabeledInput label="Optimaliseringsfaktor" value={energy.optimizationFactor} onChange={v=> setEnergy({...energy, optimizationFactor:Number(v)||1})} />
        </div>
        <p className="text-xs text-neutral-500 mt-2">Tips: Sett faktor 0.85–0.9 for 10–15% kutt med styrt ventilasjon/lys.</p>
      </Card>
    </section>
  );
}

function LabeledInput({ label, value, onChange }) {
  return (
    <label className="flex flex-col gap-1">
      <span className="text-neutral-600">{label}</span>
      <input className="px-3 py-2 rounded border" value={value} onChange={e=>onChange(e.target.value)} />
    </label>
  );
}

// ----- Admin -----
function AdminView({ rooms, setRooms, bookings, dateISO, pricing, setPricing }) {
  const addRoom = (type) => { const idx = rooms.filter(r=>r.type===type).length+1; setRooms([...rooms, { id: nextId(), name: `${typeLabel(type)} ${idx}`, type }]); };
  const removeRoom = (id) => setRooms(rooms.filter(r=>r.id!==id));
  const totalHours = rooms.length * HOURS_PER_DAY;
  const booked = Object.values(bookings[dateISO]||{}).reduce((acc, byHour) => acc + Object.keys(byHour).length, 0);

  const [gStandard, gKultSkole, gKultEnhet] = [
    pricing.groups.standard,
    pricing.groups.kulturskole,
    pricing.groups.kulturenheten,
  ];

  return (
    <section className="mt-6 grid md:grid-cols-2 gap-4">
      <Card title="Rom i systemet">
        <ul className="divide-y">
          {rooms.map(r => (
            <li key={r.id} className="py-2 flex items-center justify-between">
              <span><b>{r.name}</b> <span className="text-xs text-neutral-500">• {roomTypeLabel(r.type)}</span></span>
              <button className="px-2 py-1 text-sm rounded border" onClick={()=>removeRoom(r.id)}>Slett</button>
            </li>
          ))}
        </ul>
        <div className="flex gap-2 mt-3">
          <button className="px-3 py-2 rounded border" onClick={()=>addRoom("solo")}>+ Solo</button>
          <button className="px-3 py-2 rounded border" onClick={()=>addRoom("band")}>+ Band</button>
          <button className="px-3 py-2 rounded border" onClick={()=>addRoom("preprod")}>+ Preprod</button>
        </div>
      </Card>
      <Card title="Gruppepriser (multiplikator)">
        <div className="grid grid-cols-2 gap-3 text-sm">
          <LabeledInput label="Standard" value={gStandard} onChange={v=> setPricing(curr=> ({...curr, groups:{...curr.groups, standard: Number(v)||1}}))} />
          <LabeledInput label="Kulturskole" value={gKultSkole} onChange={v=> setPricing(curr=> ({...curr, groups:{...curr.groups, kulturskole: Number(v)||0.7}}))} />
          <LabeledInput label="Kulturenheten" value={gKultEnhet} onChange={v=> setPricing(curr=> ({...curr, groups:{...curr.groups, kulturenheten: Number(v)||0.75}}))} />
        </div>
        <div className="text-xs text-neutral-500 mt-2">Sluttpriser i dag: solo {formatNOK(computePrice('solo',pricing,'standard'))} / {formatNOK(computePrice('solo',pricing,'kulturskole'))} / {formatNOK(computePrice('solo',pricing,'kulturenheten'))},
          band {formatNOK(computePrice('band',pricing,'standard'))} / {formatNOK(computePrice('band',pricing,'kulturskole'))} / {formatNOK(computePrice('band',pricing,'kulturenheten'))}.</div>
      </Card>
      <Card title="Status i dag (for kontroll)">
        <div className="text-sm grid grid-cols-2 gap-2">
          <div className="p-3 rounded bg-neutral-50 border">Total timer: <b>{totalHours}</b></div>
          <div className="p-3 rounded bg-neutral-50 border">Booket timer: <b>{booked}</b></div>
          <div className="p-3 rounded bg-neutral-50 border">Utnyttelse: <b>{((booked/totalHours)*100).toFixed(1)}%</b></div>
          <div className="p-3 rounded bg-neutral-50 border">Dato: <b>{dateISO}</b></div>
        </div>
        <div className="mt-3">
          <button className="px-3 py-2 rounded bg-white border mr-2" onClick={()=>{ localStorage.clear(); window.location.reload(); }}>Nullstill all demo-data</button>
        </div>
        <p className="text-xs text-neutral-500 mt-2">NB: Rom/vouchers/energi/priser lagres lokalt. Bookinger bruker Supabase når konfigurert.</p>
      </Card>
    </section>
  );
}

// ----- Footer -----
function Footer(){
  return (
    <footer className="mt-auto border-t border-neutral-200 bg-white/80">
      <div className="mx-auto max-w-7xl px-4 py-4 text-sm text-neutral-600">
        Øvingsrommet.
      </div>
    </footer>
  );
}

// ----- Access Modal -----
function AccessModal({ open, onClose, grant, booking, error, loading }) {
  if (!open) return null;
  return (
    <div className="fixed inset-0 bg-black/30 flex items-center justify-center p-4 z-50">
      <div className="bg-white rounded-xl p-4 w-full max-w-md shadow-lg">
        <div className="flex justify-between items-center mb-2">
          <h3 className="font-semibold">{booking ? `Tilgang til ${booking.roomName} ${fmtHour(booking.hour)}–${fmtHour(booking.hour+1)}` : 'Tilgang'}</h3>
          <button className="text-sm" onClick={onClose}>Lukk</button>
        </div>
        {loading && <div className="text-sm text-neutral-600">Utsteder nøkkel…</div>}
        {error && <div className="text-sm text-red-600">Feil: {error}</div>}
        {grant && (
          <div className="space-y-2">
            {(grant.start_at || grant.end_at) && (
              <div className="text-sm">Gyldig: {grant.start_at?.replace('T',' ').slice(0,16)} – {grant.end_at?.replace('T',' ').slice(0,16)}</div>
            )}
            {grant.secret && (
              <div className="text-center">
                <div className="text-xs text-neutral-500">PIN-kode</div>
                <div className="text-3xl font-mono tracking-widest">{grant.secret}</div>
              </div>
            )}
            {grant.deep_link && (
              <a href={grant.deep_link} target="_blank" className="inline-block px-3 py-2 rounded bg-neutral-900 text-white text-sm">Åpne dør</a>
            )}
            {!grant.secret && !grant.deep_link && <div className="text-sm text-neutral-600">Nøkkel utstedt.</div>}
          </div>
        )}
      </div>
    </div>
  );
}


// ----- Supabase mapping helpers -----
async function refreshBookings(dateISO, setBookings) {
  const { data, error } = await supabase
    .from('bookings')
    .select('id, date, room_id, hour, type, room_name, created_by, voucher_partner, booked_for, group_code, price_nok')
    .eq('date', dateISO)
    .order('room_id')
    .order('hour');
  if (error) throw error;
  const byRoom = {};
  for (const r of data) {
    byRoom[r.room_id] = byRoom[r.room_id] || {};
    byRoom[r.room_id][r.hour] = {
      id: r.id,
      date: r.date,
      roomId: r.room_id,
      hour: r.hour,
      type: r.type,
      typeLabel: roomTypeLabel(r.type),
      roomName: r.room_name,
      createdBy: r.created_by,
      voucherPartner: r.voucher_partner || null,
      bookedFor: r.booked_for || null,
      groupCode: r.group_code || 'standard',
      priceNOK: typeof r.price_nok === 'number' ? r.price_nok : undefined,
    };
  }
  setBookings({ [dateISO]: byRoom });
}

async function refreshRange(startISO, endISO){
  const { data, error } = await supabase
    .from('bookings')
    .select('date, room_id, hour')
    .gte('date', startISO)
    .lte('date', endISO);
  if (error) throw error;
  const byDate = {};
  for (const r of data) {
    byDate[r.date] = byDate[r.date] || {};
    byDate[r.date][r.room_id] = byDate[r.date][r.room_id] || {};
    byDate[r.date][r.room_id][r.hour] = true;
  }
  return byDate;
}

function filterRangeLocal(store, startISO, endISO){
  const out={};
  for (const d of Object.keys(store||{})){
    if (d>=startISO && d<=endISO){ out[d] = store[d]; }
  }
  return out;
}

function computeUtilizationRange(rangeMap, rooms, startISO, endISO){
  const days = eachDateISO(startISO, endISO).length;
  const total = rooms.length * HOURS_PER_DAY * days;
  let booked = 0;
  for (const d of Object.keys(rangeMap||{})){
    const byRoom = rangeMap[d] || {};
    for (const rid of Object.keys(byRoom)) booked += Object.keys(byRoom[rid]||{}).length;
  }
  const utilization = total? (booked/total)*100 : 0;
  return { utilization, booked, total };
}

// ----- Stats / Business Logic (pure) -----
function computeStats({ bookings, dateISO, rooms, energy }) {
  const hours = hoursArray();
  const day = bookings[dateISO] || {};
  const totalSlots = rooms.length * hours.length;
  let bookedCount = 0; let revenue = 0; const todayList = [];
  for (const room of rooms) {
    const byHour = day[room.id] || {};
    for (const h of hours) { if (byHour[h]) { bookedCount++; const item=byHour[h]; const p = typeof item.priceNOK==='number' ? item.priceNOK : (RATECARD[room.type]||0); revenue += p; todayList.push(item); } }
  }
  const utilization = totalSlots ? (bookedCount / totalSlots) * 100 : 0;
  const byTypeCounts = todayList.reduce((acc, b) => { acc[b.type] = (acc[b.type]||0)+1; return acc; }, {});
  const totalBooked = todayList.length || 1;
  const baselineKwhPerHour = ((byTypeCounts["solo"]||0)*energy.solo + (byTypeCounts["band"]||0)*energy.band + (byTypeCounts["preprod"]||0)*energy.preprod) / totalBooked || 0;
  return { utilization, revenueToday: revenue, kwhPerBookedHour: baselineKwhPerHour, kwhOptimizedPerHour: baselineKwhPerHour * energy.optimizationFactor, energy, todayList };
}

function computeBaselineEnergy(stats) {
  const byType = stats.todayList.reduce((acc, b)=>{ acc[b.type] = (acc[b.type]||0)+1; return acc; }, {});
  const total = stats.todayList.length || 1;
  const base = ((byType.solo||0)*stats.energy.solo + (byType.band||0)*stats.energy.band + (byType.preprod||0)*stats.energy.preprod)/ total;
  return base || (stats.energy.solo+stats.energy.band+stats.energy.preprod)/3;
}

// Local store helpers (for fallback and tests)
function addBooking(bookings, b) { const out = clone(bookings); out[b.date]=out[b.date]||{}; out[b.date][b.roomId]=out[b.date][b.roomId]||{}; if (out[b.date][b.roomId][b.hour]) return bookings; out[b.date][b.roomId][b.hour]=b; return out; }
function removeBooking(bookings, b) { const out = clone(bookings); const cell = out[b.date]?.[b.roomId]?.[b.hour]; if (!cell) return bookings; delete out[b.date][b.roomId][b.hour]; return out; }

function exportCSV(list) {
  const header = ["Dato","Rom","Type","Gruppe","Start","Slutt","Tilgangskode","Pris (NOK)"];
  const rows = list.map(b => [b.date, b.roomName, roomTypeLabel(b.type), b.groupCode||'standard', fmtHour(b.hour), fmtHour(b.hour+1), generateAccessCode(b), typeof b.priceNOK==='number'? b.priceNOK : (RATECARD[b.type]||0)]);
  const csv = [header, ...rows].map(r => r.map(x => `"${String(x).replace(/"/g,'""')}"`).join(",")).join("\n");
  const blob = new Blob(["\ufeff"+csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob); const a = document.createElement("a"); a.href = url; a.download = `bookinger_${todayISO()}.csv`; a.click(); URL.revokeObjectURL(url);
}

// ----- Utils -----
function fmtHour(h) { return `${String(h).padStart(2,"0")}:00`; }
function roomTypeLabel(t) { return t === "solo" ? "Solo" : t === "band" ? "Band" : "Preprod"; }
function typeLabel(t){ return t.charAt(0).toUpperCase()+t.slice(1); }
function formatNOK(n){ try { return new Intl.NumberFormat('nb-NO', { style:'currency', currency:'NOK', maximumFractionDigits:0 }).format(n); } catch { return `${Math.round(n)} kr`; } }

// ----- Self Tests (console.assert) -----
function runSelfTests() {
  try {
    // Test 1: Access code deterministic
    const b = { date: '2025-09-15', roomId: 's1', hour: 10 };
    const c1 = generateAccessCode(b), c2 = generateAccessCode(b);
    console.assert(c1 === c2, 'Access code must be deterministic for same input');

    // Test 2: Add booking then prevent double-booking (local pure store)
    let store = {};
    store = addBooking(store, { ...b, type: 'solo', typeLabel: 'Solo', roomName: 'Solo 1' });
    const afterDouble = addBooking(store, { ...b, type: 'solo', typeLabel: 'Solo', roomName: 'Solo 1' });
    console.assert(JSON.stringify(store) === JSON.stringify(afterDouble), 'Double-booking should not change store');

    // Test 3: Remove booking
    const beforeRemove = JSON.stringify(store);
    store = removeBooking(store, b);
    console.assert(JSON.stringify(store) !== beforeRemove, 'Remove should change store');

    // Test 4: Stats utilization for one booking in a single-room, 1-hour day
    const oneRoom = [{ id:'r1', name:'R', type:'solo' }];
    const fake = { '2025-09-15': { r1: { 10: { date:'2025-09-15', roomId:'r1', hour:10, type:'solo', typeLabel:'Solo', roomName:'R', groupCode:'kulturskole', priceNOK: computePrice('solo', DEFAULT_PRICING, 'kulturskole') } } } };
    const st = computeStats({ bookings: fake, dateISO: '2025-09-15', rooms: oneRoom, energy: DEFAULT_ENERGY });
    const expectedUtil = (1 / HOURS_PER_DAY) * 100;
    console.assert(Math.abs(st.utilization - expectedUtil) < 0.001, 'Utilization must be 1 booked out of day slots');

    // Test 5: Price computation
    const soloStd = computePrice('solo', DEFAULT_PRICING, 'standard');
    const soloKS = computePrice('solo', DEFAULT_PRICING, 'kulturskole');
    console.assert(soloStd === 199 && soloKS === Math.round(199*0.7), 'Group pricing should apply multiplier');

    // Test 6: Baseline vs optimized energy
    const rooms2 = [{ id:'r1', name:'Solo', type:'solo' }, { id:'r2', name:'Band', type:'band' }];
    const fake2 = { '2025-09-15': { r1: { 10: { date:'2025-09-15', roomId:'r1', hour:10, type:'solo', typeLabel:'Solo', roomName:'Solo' } }, r2: { 10: { date:'2025-09-15', roomId:'r2', hour:10, type:'band', typeLabel:'Band', roomName:'Band' } } } };
    const st2 = computeStats({ bookings: fake2, dateISO: '2025-09-15', rooms: rooms2, energy: DEFAULT_ENERGY });
    const expectedBaseline = (DEFAULT_ENERGY.solo + DEFAULT_ENERGY.band) / 2;
    console.assert(Math.abs(st2.kwhPerBookedHour - expectedBaseline) < 1e-9, 'Baseline kWh/h must match average of types');

    // Test 7: Booking mode logic
    console.assert(determineBookingMode({voucherRequired:true, bookForOthers:false})==='voucher', 'Mode should be voucher');
    console.assert(determineBookingMode({voucherRequired:false, bookForOthers:true})==='external', 'Mode should be external');
    console.assert(determineBookingMode({voucherRequired:false, bookForOthers:false})==='open', 'Mode should be open');

    // Test 8: Week range utilization (syntetisk)
    const rmap = {};
    const monday = '2025-09-15'; // mandag
    rmap[monday] = { r1: { 10: true } }; // én time booket av totalt 13*7
    const wk = computeUtilizationRange(rmap, [{id:'r1'}], startOfWeekISO(monday), endOfWeekISO(monday));
    const wkExpected = (1 / (1 * HOURS_PER_DAY * 7)) * 100;
    console.assert(Math.abs(wk.utilization - wkExpected) < 1e-6, 'Week utilization should match');

    // Test 9: Pris for kulturenheten og avrunding
    const bandStd = computePrice('band', DEFAULT_PRICING, 'standard');
    const bandKE = computePrice('band', DEFAULT_PRICING, 'kulturenheten');
    console.assert(bandStd === 399 && bandKE === Math.round(399*0.75), 'Kulturenheten-pris skal bruke 0.75-multiplier');

    // Test 10: Månedsutnyttelse over flere dager
    const rmapM = { '2025-09-01': { r1: { 10: true } }, '2025-09-03': { r1: { 12: true, 13: true } } };
    const mStats = computeUtilizationRange(rmapM, [{id:'r1'}], '2025-09-01', '2025-09-07');
    const mBooked = 3; const mTotal = 1 * HOURS_PER_DAY * 7;
    console.assert(Math.abs(mStats.utilization - (mBooked/mTotal*100)) < 1e-6, 'Month range utilization should match synthetic data');

    console.info('%cSelf-tests passed', 'color: green');
  } catch (e) {
    console.error('Self-tests error:', e);
  }
}
