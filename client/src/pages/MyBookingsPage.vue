<script setup>
import { ref, computed, onMounted } from 'vue'
import { useGlobalStore } from '../stores/global.js'
import {
  getMyBookingsUser,
  getMyBookingsGuest,
  cancelBookingUser,
  cancelBookingGuest,
  rescheduleBookingUser,
  searchFlights,
  getPassengersByBooking,
  getSeatById,
  getMyPassengers,
  getPassengerById,
  getSeatsByFlight,
} from '../api.js'

const globalStore = useGlobalStore()
const isLoggedIn  = computed(() => !!globalStore.user.token)

// ── State ──────────────────────────────────────────────────────────────────
const bookings        = ref([])
const loading         = ref(false)
const hasSearched     = ref(false)
const errorMsg        = ref('')
const guestEmailInput = ref('')
const cancellingRef   = ref('')

// ── Expanded detail panels (Set of booking._id) ────────────────────────────
const openDetails = ref(new Set())

function toggleDetails(id) {
  const next = new Set(openDetails.value)
  next.has(id) ? next.delete(id) : next.add(id)
  openDetails.value = next
}

// ── Rebook modal state ─────────────────────────────────────────────────────
const rebookTarget      = ref(null)   // booking being rebooked
const rebookDate        = ref('')
const rebookFlights     = ref([])
const rebookLoading     = ref(false)
const rebookSubmitting  = ref(false)
const rebookError       = ref('')
const selectedFlightId  = ref('')
const rebookSeats       = ref([])     // seats for the chosen flight
const seatsLoading      = ref(false)
const selectedSeatId    = ref('')

function openRebookModal(booking) {
  rebookTarget.value     = booking
  rebookDate.value       = ''
  rebookFlights.value    = []
  rebookLoading.value    = false
  rebookSubmitting.value = false
  rebookError.value      = ''
  selectedFlightId.value = ''
  rebookSeats.value      = []
  seatsLoading.value     = false
  selectedSeatId.value   = ''
}
function closeRebookModal() {
  rebookTarget.value = null
}

async function searchRebookFlights() {
  if (!rebookDate.value) { rebookError.value = 'Please pick a date.'; return }
  rebookLoading.value    = true
  rebookError.value      = ''
  rebookFlights.value    = []
  selectedFlightId.value = ''
  rebookSeats.value      = []
  selectedSeatId.value   = ''
  try {
    const b      = rebookTarget.value
    const origin = b.flightId?.originAirportId?._id      || b.flightId?.originAirportId
    const dest   = b.flightId?.destinationAirportId?._id || b.flightId?.destinationAirportId
    const res    = await searchFlights(origin, dest, rebookDate.value)
    console.log('DEBUG - searchFlights response:', res)
    rebookFlights.value = res?.result || res?.flights || (Array.isArray(res) ? res : [])
    if (rebookFlights.value.length === 0) {
      rebookError.value = 'No flights available on that date for this route.'
    }
  } catch (err) {
    rebookError.value = err.response?.data?.message || 'Could not fetch available flights.'
  } finally {
    rebookLoading.value = false
  }
}

// Called when user clicks a flight row — fetches available seats for that flight
async function selectRebookFlight(flightId) {
  selectedFlightId.value = flightId
  selectedSeatId.value   = ''
  rebookSeats.value      = []
  seatsLoading.value     = true
  rebookError.value      = ''
  try {
    const res = await getSeatsByFlight(flightId)
    rebookSeats.value = (res?.result || res?.seats || []).filter(s => s.isAvailable !== false)
  } catch (err) {
    rebookError.value = 'Could not load seats for this flight.'
  } finally {
    seatsLoading.value = false
  }
}

async function confirmRebook() {
  if (!selectedFlightId.value) { rebookError.value = 'Please select a flight.'; return }
  if (!selectedSeatId.value)   { rebookError.value = 'Please select a seat.'; return }
  rebookSubmitting.value = true
  rebookError.value      = ''
  try {
    await rescheduleBookingUser(rebookTarget.value.bookingReference, {
      newFlightId: selectedFlightId.value,
      newSeatId:   selectedSeatId.value
    })
    closeRebookModal()
    await loadBookings()
  } catch (err) {
    rebookError.value = err.response?.data?.message || 'Rebook failed. Please try again.'
  } finally {
    rebookSubmitting.value = false
  }
}

// ── Helpers ────────────────────────────────────────────────────────────────
function formatTime(dt) {
  if (!dt) return '—'
  return new Date(dt).toLocaleString('en-PH', {
    timeZone: 'Asia/Manila', hour: '2-digit', minute: '2-digit'
  })
}
function formatDateLabel(dt) {
  if (!dt) return ''
  return new Date(dt).toLocaleDateString('en-PH', {
    weekday: 'short', day: 'numeric', month: 'short', year: 'numeric'
  })
}
function formatTimeOnly(dt) {
  if (!dt) return '—'
  return new Date(dt).toLocaleTimeString('en-PH', {
    timeZone: 'Asia/Manila', hour: '2-digit', minute: '2-digit'
  })
}
function calcTravelTime(departure, arrival) {
  if (!departure || !arrival) return ''
  const diff = new Date(arrival) - new Date(departure)
  if (diff <= 0) return ''
  const h = Math.floor(diff / 3600000)
  const m = Math.floor((diff % 3600000) / 60000)
  return `${h}h ${m}m`
}
function statusBadgeClass(status) {
  if (status === 'cancelled') return 'fc-badge cancelled'
  if (status === 'pending')   return 'fc-badge pending'
  return 'fc-badge success'
}
function seatClassLabel(cls) {
  if (!cls) return ''
  return cls.charAt(0).toUpperCase() + cls.slice(1)
}
function isUpcoming(booking) {
  const dep = booking.flightId?.departureTime
  return dep ? new Date(dep) > new Date() : true
}

// ── Enrich a single booking with passenger + seat data ────────────────────
// passengerMap: Map<passengerId string → passenger object> built before enriching.
// The BookingPassenger record stores a passengerId ref — name lives on the Passenger doc.
async function enrichBooking(b, passengerMap = new Map()) {
  try {
    const bkpRes = await getPassengersByBooking(b._id)
    const bkp    = bkpRes?.result?.find(r => r.isActive) ?? bkpRes?.result?.[0]
    if (!bkp) return b

    b.ticketNumber = bkp.ticketNumber ?? null

    // Resolve passenger name: try the pre-built map first (O(1)),
    // then fall back to a direct fetch if the map missed (e.g. guest flow).
    const passengerId = bkp.passengerId?._id
      ? String(bkp.passengerId._id)
      : bkp.passengerId
        ? String(bkp.passengerId)
        : null

    if (passengerId) {
      let passenger = passengerMap.get(passengerId) ?? null

      if (!passenger) {
        // Not in map (guest flow or edge case) — fetch directly
        try {
          const pRes = await getPassengerById(passengerId)
          passenger  = pRes?.result ?? null
        } catch { /* leave null */ }
      }

      if (passenger) {
        b.passengerName = [passenger.firstName, passenger.lastName]
          .filter(Boolean).join(' ') || null
      }
    }

    if (bkp.seatId) {
      try {
        const seatRes = await getSeatById(bkp.seatId)
        b.seat = seatRes?.result ?? null
      } catch { b.seat = null }
    }
  } catch { /* enrichment failed silently — show — in UI */ }
  return b
}

// ── Load bookings ──────────────────────────────────────────────────────────
async function loadBookings() {
  loading.value     = true
  errorMsg.value    = ''
  openDetails.value = new Set()
  try {
    const res = await getMyBookingsUser()
    console.log('DEBUG - Full response from backend:', res)
    const raw = res?.bookings || res?.result || (Array.isArray(res) ? res : [])

    // Pre-build passenger map so enrichBooking doesn't fetch each one individually
    const passengerMap = new Map()
    try {
      const pRes = await getMyPassengers()
      for (const p of pRes?.passengers ?? pRes?.result ?? []) {
        passengerMap.set(String(p._id), p)
      }
    } catch { /* map stays empty; enrichBooking will fall back to getPassengerById */ }

    bookings.value = await Promise.all(raw.map(b => enrichBooking(b, passengerMap)))
  } catch (err) {
    console.error('CATCH TARGET EXCEPTION LOG:', err)
    if (err.response?.status === 404) {
      bookings.value = []
    } else {
      errorMsg.value = err.response?.data?.message || 'Could not load your bookings right now.'
    }
  } finally {
    loading.value     = false
    hasSearched.value = true
  }
}

async function lookupGuestBookings() {
  if (!guestEmailInput.value || !guestEmailInput.value.includes('@')) {
    errorMsg.value = 'Please enter a valid email address.'
    return
  }
  loading.value     = true
  errorMsg.value    = ''
  openDetails.value = new Set()
  try {
    const res = await getMyBookingsGuest({ guestEmail: guestEmailInput.value })
    console.log('DEBUG - Guest bookings API response:', res)
    const raw      = res?.result || res?.data || (Array.isArray(res) ? res : [])
    bookings.value = await Promise.all(raw.map(b => enrichBooking(b, new Map())))
  } catch (err) {
    console.error(err)
    bookings.value = []
    errorMsg.value = err.response?.data?.message || 'No bookings found for that email.'
  } finally {
    loading.value     = false
    hasSearched.value = true
  }
}

async function cancelBooking(booking) {
  if (!window.confirm(`Cancel booking ${booking.bookingReference}? This cannot be undone.`)) return
  cancellingRef.value = booking.bookingReference
  errorMsg.value      = ''
  try {
    if (isLoggedIn.value) {
      await cancelBookingUser(booking.bookingReference)
    } else {
      await cancelBookingGuest(booking.bookingReference, { guestEmail: guestEmailInput.value })
    }
    booking.status = 'cancelled'
  } catch (err) {
    errorMsg.value = err.response?.data?.message || 'Could not cancel this booking.'
  } finally {
    cancellingRef.value = ''
  }
}

onMounted(() => {
  if (isLoggedIn.value) loadBookings()
})

</script>

<template>
  <div class="page active">
    <div class="inner-page">
      <div class="pt-5">
        <div class="container mt-5" style="max-width:900px;">

          <nav class="theme-breadcrumb" aria-label="breadcrumb">
            <ol class="breadcrumb mb-0">
              <li class="breadcrumb-item"><RouterLink :to="{ name: 'Home' }">Home</RouterLink></li>
              <li class="breadcrumb-item active">My Bookings</li>
            </ol>
          </nav>

          <h1 class="confirm-headline mb-4">My <em class="gold">Bookings</em></h1>

          <!-- Guest lookup -->
          <div v-if="!isLoggedIn" class="booking-section mb-4">
            <div class="bs-body">
              <label class="f-label">Find your bookings by email</label>
              <div class="row g-2">
                <div class="col-8 col-md-9">
                  <input
                    type="email"
                    class="f-input"
                    v-model="guestEmailInput"
                    placeholder="you@email.com"
                    @keyup.enter="lookupGuestBookings"
                  />
                </div>
                <div class="col-4 col-md-3">
                  <button class="fc-select-btn w-100" @click="lookupGuestBookings">Find</button>
                </div>
              </div>
              <p class="body-text mt-2" style="font-size:0.78rem;">
                Booked while logged in?
                <RouterLink :to="{ name: 'Login', query: { redirect: '/my-bookings' } }" class="gold-link">Log in</RouterLink>
                to see everything in one place.
              </p>
            </div>
          </div>

          <div v-if="errorMsg" class="alert alert-danger">{{ errorMsg }}</div>

          <div v-if="loading" class="text-center py-5">
            <span class="spinner-border text-warning"></span>
          </div>

          <div v-else-if="hasSearched && bookings.length === 0" class="text-center py-5">
            <p class="body-text">No bookings found.</p>
            <RouterLink :to="{ name: 'SearchFlights' }" class="gold-link">Search for a flight →</RouterLink>
          </div>

          <!-- ── Booking cards ──────────────────────────────────────────── -->
          <div
            v-for="booking in bookings"
            :key="booking._id"
            class="booking-card-wrap mb-3"
            :class="{ 'is-cancelled': booking.status === 'cancelled' }"
          >
            <!-- Flight row -->
            <div class="flight-card" style="cursor:default; margin-bottom:0; border-radius:0;">

              <div class="fc-endpoint">
                <div class="fc-time">{{ formatTime(booking.flightId?.departureTime) }}</div>
                <div class="fc-airport">
                  {{ booking.flightId?.originAirportId?.city
                  || booking.flightId?.originAirportId?.iataCode || 'DEP' }}
                </div>
                <div class="fc-date">{{ formatDateLabel(booking.flightId?.departureTime) }}</div>
              </div>

              <div class="fc-mid">
                <div class="fc-duration">
                  {{ calcTravelTime(booking.flightId?.departureTime, booking.flightId?.arrivalTime) }}
                </div>
                <div class="fc-line">
                  <span class="fc-plane-icon"><i class="bi bi-airplane-fill"></i></span>
                </div>
                <div class="fc-stops">{{ booking.flightId?.airlineId?.name || '' }}</div>
              </div>

              <div class="fc-endpoint">
                <div class="fc-time">{{ formatTime(booking.flightId?.arrivalTime) }}</div>
                <div class="fc-airport">
                  {{ booking.flightId?.destinationAirportId?.city
                  || booking.flightId?.destinationAirportId?.iataCode || 'ARR' }}
                </div>
                <div class="fc-date">{{ formatDateLabel(booking.flightId?.arrivalTime) }}</div>
              </div>

              <!-- Price / status / action panel -->
              <div
                class="fc-price-box d-flex flex-column align-items-stretch justify-content-start text-end"
                style="min-width:160px; gap:4px;"
              >
                <div>
                  <span :class="statusBadgeClass(booking.status)">{{ booking.status }}</span>
                </div>
                <div class="fc-price-amt mt-1" style="font-size:1.2rem; font-weight:700;">
                  ₱{{ booking.totalAmount?.toLocaleString() }}
                </div>
                <div class="fc-price-note text-warning" style="font-size:0.75rem;">
                  Ref: {{ booking.bookingReference }}
                </div>
                <div v-if="booking.checkedIn" class="fc-price-note text-success" style="font-size:0.75rem;">
                  <i class="bi bi-check-circle-fill"></i> Checked in
                </div>

                <div class="d-flex flex-column gap-2 mt-2 w-100">

                  <!-- Check-in -->
                  <RouterLink
                    v-if="booking.status === 'confirmed' && !booking.checkedIn && isUpcoming(booking)"
                    :to="{ name: 'CheckIn', query: { ref: booking.bookingReference } }"
                    class="fc-select-btn d-block text-center text-decoration-none w-100"
                    style="padding:6px 0;"
                  >Check-in</RouterLink>

                  <!-- Rebook -->
                  <button
                    v-if="booking.status !== 'cancelled' && isUpcoming(booking)"
                    class="fc-select-btn d-block w-100"
                    style="padding:6px 0;"
                    @click="openRebookModal(booking)"
                  >Rebook</button>

                  <!-- Cancel -->
                  <button
                    v-if="booking.status !== 'cancelled' && !booking.checkedIn"
                    class="fc-select-btn d-block w-100"
                    style="background:transparent; color:#ff4d4d; border:1px solid #ff4d4d; padding:6px 0;"
                    :disabled="cancellingRef === booking.bookingReference"
                    @click="cancelBooking(booking)"
                  >{{ cancellingRef === booking.bookingReference ? 'Cancelling…' : 'Cancel' }}</button>

                  <!-- Toggle details -->
                  <button class="bk-details-toggle w-100" @click="toggleDetails(booking._id)">
                    <i :class="`bi ${openDetails.has(booking._id) ? 'bi-chevron-up' : 'bi-chevron-down'} me-1`"></i>
                    {{ openDetails.has(booking._id) ? 'Less info' : 'More info' }}
                  </button>

                </div>
              </div>
            </div>
            <!-- /flight row -->

            <!-- ── Detail panel ──────────────────────────────────────── -->
            <div v-if="openDetails.has(booking._id)" class="bk-detail-panel">

              <div class="bk-detail-grid">

                <div class="bk-detail-cell">
                  <span class="bk-detail-label">Flight</span>
                  <span class="bk-detail-value">
                    {{ booking.flightId?.flightNumber || '—' }}
                    <span v-if="booking.flightId?.airlineId?.name" class="bk-detail-sub">
                      · {{ booking.flightId.airlineId.name }}
                    </span>
                  </span>
                </div>

                <div class="bk-detail-cell">
                  <span class="bk-detail-label">Passenger</span>
                  <span class="bk-detail-value">{{ booking.passengerName || '—' }}</span>
                </div>

                <div class="bk-detail-cell">
                  <span class="bk-detail-label">Seat</span>
                  <span class="bk-detail-value">
                    <template v-if="booking.seat">
                      {{ booking.seat.seatNumber }}
                      <span class="bk-seat-badge" :class="booking.seat.class">
                        {{ seatClassLabel(booking.seat.class) }}
                      </span>
                    </template>
                    <template v-else>—</template>
                  </span>
                </div>

                <div class="bk-detail-cell">
                  <span class="bk-detail-label">Ticket No.</span>
                  <span class="bk-detail-value font-monospace" style="font-size:0.82rem;">
                    {{ booking.ticketNumber || '—' }}
                  </span>
                </div>

                <div class="bk-detail-cell">
                  <span class="bk-detail-label">Departure Terminal</span>
                  <span class="bk-detail-value">
                    {{ booking.flightId?.originTerminal ? 'Terminal ' + booking.flightId.originTerminal : '—' }}
                  </span>
                </div>

                <div class="bk-detail-cell">
                  <span class="bk-detail-label">Arrival Terminal</span>
                  <span class="bk-detail-value">
                    {{ booking.flightId?.destinationTerminal ? 'Terminal ' + booking.flightId.destinationTerminal : '—' }}
                  </span>
                </div>

              </div>

              <!-- Full dep / arr time block -->
              <div class="bk-times-row">
                <div class="bk-time-block">
                  <span class="bk-detail-label">Departure</span>
                  <span class="bk-time-big">{{ formatTimeOnly(booking.flightId?.departureTime) }}</span>
                  <span class="bk-time-date">{{ formatDateLabel(booking.flightId?.departureTime) }}</span>
                </div>
                <div class="bk-time-arrow"><i class="bi bi-arrow-right"></i></div>
                <div class="bk-time-block">
                  <span class="bk-detail-label">Arrival</span>
                  <span class="bk-time-big">{{ formatTimeOnly(booking.flightId?.arrivalTime) }}</span>
                  <span class="bk-time-date">{{ formatDateLabel(booking.flightId?.arrivalTime) }}</span>
                </div>
              </div>

            </div>
            <!-- /detail panel -->

          </div>
          <!-- /booking cards -->

        </div>
      </div>
    </div>
  </div>

  <!-- ════════════════════════════════════════════════════════════════ -->
  <!-- REBOOK MODAL                                                    -->
  <!-- ════════════════════════════════════════════════════════════════ -->
  <teleport to="body">
    <div v-if="rebookTarget" class="rb-overlay" @click.self="closeRebookModal">
      <div class="rb-modal" role="dialog" aria-modal="true" aria-labelledby="rb-title">

        <!-- Header -->
        <div class="rb-header">
          <h5 id="rb-title" class="mb-0">
            Rebook Flight
            <span class="gold"> · {{ rebookTarget.bookingReference }}</span>
          </h5>
          <button class="rb-close" @click="closeRebookModal" aria-label="Close">
            <i class="bi bi-x-lg"></i>
          </button>
        </div>

        <!-- Current route summary -->
        <div class="rb-current-flight">
          <span class="rb-route">
            {{ rebookTarget.flightId?.originAirportId?.iataCode || 'DEP' }}
            <i class="bi bi-arrow-right mx-2"></i>
            {{ rebookTarget.flightId?.destinationAirportId?.iataCode || 'ARR' }}
          </span>
          <span class="rb-current-label">
            Currently: {{ formatDateLabel(rebookTarget.flightId?.departureTime) }}
          </span>
        </div>

        <!-- Body: date picker + results -->
        <div class="rb-body">

          <label class="f-label mb-1">Select a new date</label>
          <div class="d-flex gap-2 align-items-center">
            <input
              type="date"
              class="f-input flex-grow-1"
              v-model="rebookDate"
              :min="new Date().toISOString().slice(0, 10)"
            />
            <button
              class="fc-select-btn"
              style="white-space:nowrap; padding:8px 18px;"
              :disabled="rebookLoading || !rebookDate"
              @click="searchRebookFlights"
            >
              <span v-if="rebookLoading" class="spinner-border spinner-border-sm me-1"></span>
              {{ rebookLoading ? 'Searching…' : 'Find Flights' }}
            </button>
          </div>

          <!-- Step 2: Flight options -->
          <div v-if="rebookFlights.length > 0" class="rb-flight-list mt-3">
            <label class="f-label mb-2">Choose a flight</label>
            <div
              v-for="flight in rebookFlights"
              :key="flight._id"
              class="rb-flight-option"
              :class="{ selected: selectedFlightId === flight._id }"
              @click="selectRebookFlight(flight._id)"
            >
              <div class="d-flex align-items-center gap-3 flex-wrap">
                <div class="rb-flight-times">
                  <span class="rb-ftime">{{ formatTimeOnly(flight.departureTime) }}</span>
                  <i class="bi bi-arrow-right mx-1 text-muted" style="font-size:0.8rem;"></i>
                  <span class="rb-ftime">{{ formatTimeOnly(flight.arrivalTime) }}</span>
                </div>
                <div class="rb-flight-meta">
                  {{ flight.flightNumber }}
                  <span v-if="flight.airlineId?.name"> · {{ flight.airlineId.name }}</span>
                  <span class="ms-2">{{ calcTravelTime(flight.departureTime, flight.arrivalTime) }}</span>
                </div>
                <div class="ms-auto rb-flight-price">
                  ₱{{ (flight.basePrice ?? flight.price ?? flight.economyPrice)?.toLocaleString() || '—' }}
                </div>
              </div>
              <i v-if="selectedFlightId === flight._id" class="bi bi-check-circle-fill rb-selected-tick"></i>
            </div>
          </div>

          <!-- Step 3: Seat picker (shown after a flight is selected) -->
          <div v-if="selectedFlightId" class="mt-3">
            <label class="f-label mb-2">Choose a seat</label>

            <div v-if="seatsLoading" class="text-center py-3">
              <span class="spinner-border spinner-border-sm text-warning"></span>
              <span class="ms-2" style="color:#999; font-size:0.85rem;">Loading seats…</span>
            </div>

            <div v-else-if="rebookSeats.length === 0" style="color:#999; font-size:0.85rem;">
              No available seats found for this flight.
            </div>

            <div v-else class="rb-seat-grid">
              <button
                v-for="seat in rebookSeats"
                :key="seat._id"
                class="rb-seat-btn"
                :class="{
                  selected: selectedSeatId === seat._id,
                  business: seat.class === 'business',
                  first:    seat.class === 'first'
                }"
                @click="selectedSeatId = seat._id"
              >
                <span class="rb-seat-num">{{ seat.seatNumber }}</span>
                <span class="rb-seat-cls">{{ seat.class }}</span>
              </button>
            </div>
          </div>

          <div v-if="rebookError" class="alert alert-danger mt-3 mb-0" style="font-size:0.85rem;">
            {{ rebookError }}
          </div>

        </div>

        <!-- Footer -->
        <div class="rb-footer">
          <button
            class="fc-select-btn"
            style="background:transparent; color:#888; border:1px solid #444;"
            @click="closeRebookModal"
          >Cancel</button>
          <button
            class="fc-select-btn"
            :disabled="!selectedFlightId || rebookSubmitting"
            @click="confirmRebook"
          >
            <span v-if="rebookSubmitting" class="spinner-border spinner-border-sm me-1"></span>
            {{ rebookSubmitting ? 'Confirming…' : 'Confirm Rebook' }}
          </button>
        </div>

      </div>
    </div>
  </teleport>
</template>

<style scoped>
/* ── Card wrapper ─────────────────────────────────────────────────────── */
.booking-card-wrap {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0,0,0,0.18);
  transition: opacity 0.2s;
}
.booking-card-wrap.is-cancelled { opacity: 0.6; }

/* Remove individual card radius/shadow — wrapper owns it */
.booking-card-wrap .flight-card {
  border-radius: 0 !important;
  box-shadow: none !important;
  margin-bottom: 0 !important;
}

/* ── More / Less toggle ───────────────────────────────────────────────── */
.bk-details-toggle {
  background: transparent;
  border: 1px solid rgba(255,255,255,0.13);
  border-radius: 6px;
  color: #999;
  font-size: 0.78rem;
  padding: 4px 0;
  cursor: pointer;
  transition: color 0.15s, border-color 0.15s;
}
.bk-details-toggle:hover { color: #d4a843; border-color: #d4a843; }

/* ── Detail panel ─────────────────────────────────────────────────────── */
.bk-detail-panel {
  background: #16162a;
  border-top: 1px solid rgba(212,168,67,0.18);
  padding: 20px 24px;
}
.bk-detail-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px 12px;
  margin-bottom: 20px;
}
@media (max-width: 600px) {
  .bk-detail-grid { grid-template-columns: repeat(2, 1fr); }
}
.bk-detail-cell { display: flex; flex-direction: column; gap: 3px; }
.bk-detail-label {
  font-size: 0.68rem;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  color: #777;
}
.bk-detail-value { font-size: 0.9rem; color: #e0e0e0; font-weight: 500; }
.bk-detail-sub   { color: #999; font-weight: 400; }

/* Seat class badge inside detail panel */
.bk-seat-badge {
  display: inline-block;
  font-size: 0.68rem;
  font-weight: 600;
  padding: 1px 7px;
  border-radius: 20px;
  margin-left: 5px;
  text-transform: capitalize;
}
.bk-seat-badge.economy  { background: rgba(40,167,69,0.2);  color: #4caf7d; }
.bk-seat-badge.business { background: rgba(212,168,67,0.2); color: #d4a843; }
.bk-seat-badge.first    { background: rgba(111,66,193,0.2); color: #b07eef; }

/* Departure / arrival time row */
.bk-times-row {
  display: flex;
  align-items: center;
  gap: 16px;
  padding-top: 16px;
  border-top: 1px solid rgba(255,255,255,0.07);
}
.bk-time-block { display: flex; flex-direction: column; gap: 2px; }
.bk-time-big   { font-size: 1.4rem; font-weight: 700; color: #fff; line-height: 1; }
.bk-time-date  { font-size: 0.75rem; color: #999; }
.bk-time-arrow { color: #d4a843; font-size: 1.1rem; }

/* ── Rebook overlay ───────────────────────────────────────────────────── */
.rb-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.75);
  z-index: 1050;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
}
.rb-modal {
  background: #12121f;
  border: 1px solid rgba(212,168,67,0.22);
  border-radius: 14px;
  width: 100%;
  max-width: 560px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-shadow: 0 10px 50px rgba(0,0,0,0.65);
}
.rb-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 18px 22px 14px;
  border-bottom: 1px solid rgba(255,255,255,0.08);
}
.rb-header h5 { font-size: 1rem; color: #e8e8e8; }
.rb-close {
  background: transparent; border: none; color: #aaa;
  font-size: 1rem; cursor: pointer; padding: 4px; line-height: 1;
  transition: color 0.15s;
}
.rb-close:hover { color: #fff; }

.rb-current-flight {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 6px;
  padding: 10px 22px;
  background: rgba(212,168,67,0.05);
  border-bottom: 1px solid rgba(255,255,255,0.06);
}
.rb-route        { font-size: 1rem; font-weight: 700; color: #fff; }
.rb-current-label { font-size: 0.78rem; color: #999; }

.rb-body { padding: 20px 22px; overflow-y: auto; flex: 1; }

.rb-flight-list { display: flex; flex-direction: column; gap: 8px; }
.rb-flight-option {
  position: relative;
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 8px;
  padding: 12px 14px;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
}
.rb-flight-option:hover         { border-color: rgba(212,168,67,0.4); background: rgba(212,168,67,0.04); }
.rb-flight-option.selected      { border-color: #d4a843; background: rgba(212,168,67,0.08); }
.rb-ftime       { font-weight: 700; font-size: 0.95rem; color: #e8e8e8; }
.rb-flight-meta { font-size: 0.78rem; color: #999; }
.rb-flight-price { font-size: 0.95rem; font-weight: 700; color: #d4a843; }
.rb-selected-tick {
  position: absolute; top: 10px; right: 12px;
  color: #d4a843; font-size: 1rem;
}

.rb-footer {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  padding: 14px 22px;
  border-top: 1px solid rgba(255,255,255,0.08);
}
/* ── Seat picker grid ─────────────────────────────────────────────────── */
.rb-seat-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.rb-seat-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 56px;
  height: 52px;
  border: 1px solid rgba(255,255,255,0.15);
  border-radius: 8px;
  background: rgba(255,255,255,0.04);
  color: #ccc;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s, color 0.15s;
  padding: 4px;
}
.rb-seat-btn:hover         { border-color: #d4a843; color: #d4a843; }
.rb-seat-btn.selected      { border-color: #d4a843; background: rgba(212,168,67,0.15); color: #d4a843; }
.rb-seat-btn.business      { border-color: rgba(212,168,67,0.3); }
.rb-seat-btn.first         { border-color: rgba(111,66,193,0.35); }
.rb-seat-btn.business.selected { background: rgba(212,168,67,0.18); }
.rb-seat-btn.first.selected    { background: rgba(111,66,193,0.18); color: #b07eef; border-color: #b07eef; }
.rb-seat-num { font-size: 0.85rem; font-weight: 700; line-height: 1; }
.rb-seat-cls { font-size: 0.6rem; text-transform: capitalize; opacity: 0.65; margin-top: 2px; }
</style>