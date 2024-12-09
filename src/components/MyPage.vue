<template>
  <div class="my-page-container">
    <div v-if="isLoading" class="loading-overlay">
      <div class="loading-spinner"></div>
    </div>
    <div class="my-page">
      <h1>마이 페이지</h1>
      <div class="button-container">
        <button
          v-for="section in sections"
          :key="section.value"
          @click="activeSection = section.value"
          :class="{ active: activeSection === section.value }"
        >
          {{ section.label }}
        </button>
      </div>

      <!-- 사용자 정보 섹션 -->
      <div v-if="activeSection === 'user'" class="section">
        <h2>회원 정보</h2>
        <form @submit.prevent="updateUser">
          <div class="form-group">
            <label for="nickname">닉네임</label>
            <input id="nickname" v-model="user.nickname" required>
          </div>
          <div class="form-group">
            <label for="password">비밀번호</label>
            <input id="password" v-model="user.password" type="password" required>
          </div>
          <button type="submit" class="submit-button">정보 수정</button>
        </form>
      </div>

      <!-- 쿠폰 섹션 -->
      <div v-if="activeSection === 'coupons'" class="section">
        <h2>사용 가능한 쿠폰</h2>
        <div class="coupon-list">
          <div v-for="coupon in coupons" :key="coupon.couponCode" class="coupon-item">
            <div class="coupon-main">
              <div class="coupon-header">
                <span class="coupon-name">{{ coupon.couponName }}</span>
                <span class="coupon-code">{{ coupon.couponCode }}</span>
              </div>
              <div class="coupon-details">
                <div class="discount-info">
                  {{
                    coupon.discountType === 'PERCENTAGE'
                      ? `${coupon.discount}% 할인`
                      : `${coupon.price.toLocaleString()}원 할인`
                  }}
                </div>
                <div class="coupon-type">{{ getCouponTypeText(coupon.couponType) }}</div>
              </div>
            </div>
            <div class="coupon-status" :class="{ 'inactive': !coupon.isActive }">
              {{ coupon.isActive ? '사용 가능' : '사용 불가' }}
            </div>
          </div>
        </div>
      </div>

      <!-- 예매 내역 섹션 -->
      <div v-if="activeSection === 'reservations'" class="section">
        <h2>예매 내역</h2>
        <div class="filter-buttons">
          <button @click="reservationFilter = 'all'" :class="{ active: reservationFilter === 'all' }">전체</button>
          <button @click="reservationFilter = 'pending'" :class="{ active: reservationFilter === 'pending' }">결제 대기중
          </button>
        </div>
        <div v-if="filteredReservations.length > 0" class="list-container">
          <div v-for="reservation in filteredReservations" :key="reservation.reservationId" class="list-item">
            <div>
              <span class="item-title">콘서트 ID: {{ reservation.concertId }}</span>
              <span class="item-date">{{ formatDate(reservation.createdAt) }}</span>
              <div class="item-details">
                <span>좌석: {{ reservation.seatNumber }}</span>
                <span class="item-price">{{ reservation.price.toLocaleString() }}원</span>
              </div>
              <div class="reservation-status">
                상태: {{ getReservationStatusText(reservation.status) }}
              </div>
              <button v-if="reservation.status === 'PENDING'" @click="requestTossPayment(reservation)"
                      class="payment-button">
                결제 진행
              </button>
            </div>
          </div>
        </div>
        <div v-else class="empty-message">예매 내역이 없습니다.</div>
        <div v-if="filteredReservations.length > 0" class="pagination">
          <button @click="changePage('reservations', -1)"
                  :disabled="reservationPage === 1">이전
          </button>
          <span>{{ reservationPage }} / {{ reservationTotalPages }}</span>
          <button @click="changePage('reservations', 1)" :disabled="reservationPage === reservationTotalPages">다음
          </button>
        </div>
      </div>

      <!-- 결제 내역 섹션 -->
      <div v-if="activeSection === 'payments'" class="section">
        <h2>결제 내역</h2>
        <div class="filter-buttons">
          <button @click="setPaymentFilter('all')" :class="{ active: paymentFilter === 'all' }">전체</button>
          <button @click="setPaymentFilter('PAID')" :class="{ active: paymentFilter === 'PAID' }">결제 완료</button>
          <button @click="setPaymentFilter('CANCELLED')" :class="{ active: paymentFilter === 'CANCELLED' }">결제 취소
          </button>
        </div>
        <div v-if="filteredPayments.length > 0" class="list-container">
          <div v-for="payment in filteredPayments" :key="payment.concertId" class="list-item">
            <div class="item-header">
              <span class="item-title">{{ payment.concertTitle }}</span>
              <span class="payment-status" :class="getPaymentStatusClass(payment.payStatus)">
                {{ getPaymentStatusText(payment.payStatus) }}
              </span>
            </div>
            <div class="item-details">
              <span>{{ payment.payInfo }}</span>
              <span class="item-price">{{ payment.amount.toLocaleString() }}원</span>
            </div>
          </div>
        </div>
        <div v-else class="empty-message">결제 내역이 없습니다.</div>
        <div v-if="filteredPayments.length > 0" class="pagination">
          <button @click="changePage('payments', -1)" :disabled="paymentPage === 1">이전</button>
          <span>{{ paymentPage }} / {{ paymentTotalPages }}</span>
          <button @click="changePage('payments', 1)" :disabled="paymentPage === paymentTotalPages">다음</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, computed } from 'vue';
import { useRouter, useRoute } from 'vue-router';
import axios from 'axios';

const router = useRouter();
const route = useRoute();
const activeSection = ref('user');
const user = ref({ nickname: '', password: '', id: '' });
const coupons = ref([]);
const reservations = ref([]);
const payments = ref([]);
const reservationPage = ref(1);
const reservationTotalPages = ref(1);
const reservationTotalCount = ref(0);
const paymentPage = ref(1);
const paymentTotalPages = ref(1);
const reservationFilter = ref('all');
const paymentFilter = ref('all');
const isLoading = ref(false);

const API_BASE_URL = 'https://api.jariotte.store/api/v1';

const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
  withCredentials: true
});

let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });

  failedQueue = [];
};

const refreshToken = async () => {
  try {
    const response = await api.post('/auth/refresh');
    const newToken = response.data.accessToken;
    localStorage.setItem('token', newToken);
    return newToken;
  } catch (error) {
    console.error('토큰 갱신 실패:', error);
    localStorage.removeItem('token');
    router.push('/auth/login');
    throw error;
  }
};

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers['Authorization'] = `Bearer ${token}`;
  }
  return config;
}, (error) => {
  return Promise.reject(error);
});

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({resolve, reject});
        }).then(token => {
          originalRequest.headers['Authorization'] = 'Bearer ' + token;
          return api(originalRequest);
        }).catch(err => {
          return Promise.reject(err);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const newToken = await refreshToken();
        processQueue(null, newToken);
        originalRequest.headers['Authorization'] = 'Bearer ' + newToken;
        return api(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

const tossPayments = ref(null);

const handleApiError = (error, context) => {
  console.error(`Error in ${context}:`, error);
  if (error.response) {
    console.error(error.response.data);
    console.error(error.response.status);
    console.error(error.response.headers);
  } else if (error.request) {
    console.error(error.request);
  } else {
    console.error('Error', error.message);
  }
};

const fetchData = async (fetchFunction) => {
  isLoading.value = true;
  try {
    await fetchFunction();
  } catch (error) {
    console.error('Error fetching data:', error);
  } finally {
    isLoading.value = false;
  }
};

onMounted(async () => {
  const token = localStorage.getItem('token');
  if (!token) {
    router.push('/login');
    return;
  }

  await fetchData(async () => {
    await fetchUserInfo();
    await loadTossPaymentsSDK();
    await fetchPayments();

    const {paymentKey, orderId, amount, code, message} = route.query;

    if (paymentKey && orderId && amount) {
      await handlePaymentSuccess(paymentKey, orderId, amount);
    } else if (code && message && orderId) {
      await handlePaymentFail({code, message, orderId});
    }
  });
});

const loadTossPaymentsSDK = () => {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = 'https://js.tosspayments.com/v1';
    script.onload = () => {
      if (typeof window.TossPayments !== 'undefined') {
        const clientKey = 'test_ck_vZnjEJeQVxG6k9jZ7yZO3PmOoBN0';
        tossPayments.value = window.TossPayments(clientKey);
        resolve();
      } else {
        reject(new Error('TossPayments is not defined'));
      }
    };
    script.onerror = () => reject(new Error('Failed to load Toss Payments SDK'));
    document.body.appendChild(script);
  });
};

const fetchUserInfo = async () => {
  try {
    const {data} = await api.get('/users/my');
    user.value = data.data;
  } catch (error) {
    handleApiError(error, 'fetchUserInfo');
  }
};

const updateUser = async () => {
  try {
    const {data} = await api.patch('/users/my', user.value);
    alert('회원 정보가 수정되었습니다.');
    user.value = data.data;
  } catch (error) {
    handleApiError(error, 'updateUser');
    if (error.response) {
      alert(`회원 정보 수정에 실패했습니다. 서버 응답: ${error.response.data?.message || error.response.status}`);
    } else if (error.request) {
      alert('서버에서 응답이 없습니다. 네트워크 연결을 확인해주세요.');
    } else {
      alert(`회원 정보 수정 중 오류가 발생했습니다: ${error.message}`);
    }
  }
};

const fetchCoupons = async () => {
  try {
    const {data} = await api.get('/coupons/my-coupon');
    coupons.value = data.data;
  } catch (error) {
    handleApiError(error, 'fetchCoupons');
  }
};

const fetchReservations = async () => {
  isLoading.value = true;
  try {
    const { data } = await api.get('/reservations', {
      params: { page: reservationPage.value, size: 5 },
      headers: { 'Cache-Control': 'no-cache', 'Pragma': 'no-cache' }
    });
    reservations.value = data.data.content.map(reservation => ({
      reservationId: reservation.reservationId,
      concertId: reservation.concertId,
      createdAt: reservation.createdAt,
      price: reservation.price,
      seatId: reservation.seatId,
      seatNumber: reservation.seatNumber,
      status: reservation.reservationStatus,
      userId: reservation.userId
    }));
    reservationTotalPages.value = data.data.totalPages;
    reservationTotalCount.value = data.data.totalElements;
  } catch (error) {
    handleApiError(error, 'fetchReservations');
  } finally {
    isLoading.value = false;
  }
};

const fetchPayments = async () => {
  try {
    const { data } = await api.get('/payments/my-payment', {
      params: {
        page: paymentPage.value - 1,
        size: 5,
        payStatus: paymentFilter.value === 'all' ? undefined : paymentFilter.value
      },
      headers: { 'Cache-Control': 'no-cache', 'Pragma': 'no-cache' }
    });
    const paymentsWithTitles = await Promise.all(data.data.content.map(async (payment) => {
      const concertResponse = await api.get(`/concerts/${payment.concertId}`);
      return {
        ...payment,
        concertTitle: concertResponse.data.data.title
      };
    }));
    payments.value = paymentsWithTitles;
    paymentTotalPages.value = data.data.totalPages;
  } catch (error) {
    handleApiError(error, 'fetchPayments');
  }
};


const setPaymentFilter = (filter) => {
  paymentFilter.value = filter;
  paymentPage.value = 1;
  fetchPayments();
};

const changePage = (section, delta) => {
  if (section === 'reservations') {
    reservationPage.value += delta;
    fetchReservations();
  } else if (section === 'payments') {
    paymentPage.value += delta;
    fetchPayments();
  }
};

const formatDate = (dateString) => {
  const date = new Date(dateString);
  return date.toLocaleDateString('ko-KR', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  });
};

const checkTossPayment = async (paymentKey) => {
  try {
    const tossResponse = await axios.get(
      `https://api.tosspayments.com/v1/payments/${paymentKey}`,
      {
        headers: {
          Authorization: `Basic ${btoa('test_sk_yZqmkKeP8g44q77Oj1ZK3bQRxB9l')}`
        }
      }
    );
    return tossResponse.data;
  } catch (error) {
    console.error('토스 결제 확인 실패:', error);
    return null;
  }
};

const requestTossPayment = async (reservation) => {
  try {
    if (!tossPayments.value) {
      await loadTossPaymentsSDK();
    }

    const paymentRequest = {
      reservationId: reservation.reservationId,
      amount: reservation.price,
      payInfo: `콘서트 ID ${reservation.concertId} 좌석 ${reservation.seatNumber} 예매`
    };

    const response = await api.post('/payments/toss', paymentRequest);
    console.log('Payment request response:', response.data);

    if (!response.data?.orderId) {
      throw new Error('결제 요청 정보가 올바르지 않습니다.');
    }

    await tossPayments.value.requestPayment('카드', {
      amount: response.data.amount,
      orderId: response.data.orderId,
      orderName: `콘서트 좌석 ${reservation.seatNumber} 예매`,
      customerName: user.value.nickname,
      successUrl: `${window.location.origin}/mypage`,
      failUrl: `${window.location.origin}/mypage`
    });

  } catch (error) {
    console.error('Payment request error:', error);
    let errorMessage = '결제 중 오류가 발생했습니다.';

    if (error.code === 'USER_CANCEL') {
      errorMessage = '결제가 취소되었습니다.';
    } else if (error.response?.data?.message) {
      errorMessage = error.response.data.message;
    }

    alert(errorMessage);
  }
};

const handlePaymentSuccess = async (paymentKey, orderId, amount) => {
  let tossPayment = null;

  try {
    tossPayment = await checkTossPayment(paymentKey);

    const response = await api.get('/payments/toss/success', {
      params: {
        paymentKey,
        orderId,
        amount: Number(amount)
      }
    });

    console.log('Payment success response:', response.data);

    if (tossPayment?.status === 'DONE') {
      if (response.data.includes('success')) {
        alert('결제가 성공적으로 완료되었습니다.');
        await refreshData();
      } else if (response.data.includes('error')) {
        alert('결제는 완료되었으나 처리 중 문제가 발생했습니다. 고객센터에 문의해주세요.');
      } else {
        throw new Error('알 수 없는 응답입니다.');
      }
    } else if (response.data.includes('new')) {
      alert('결제 가능 시간이 만료되었습니다. 다시 시도해주세요.');
    } else {
      throw new Error('결제가 정상적으로 처리되지 않았습니다.');
    }

    activeSection.value = 'reservations';
  } catch (error) {
    alert(errorMessage);
  } finally {
    if (route.path === '/mypage') {
      router.replace({ query: {} });
    }
  }
};

const refreshData = async () => {
  await Promise.all([
    fetchReservations(),
    fetchPayments()
  ]);
};


const handlePaymentFail = async (error) => {
  try {
    if (error.code && error.message && error.orderId) {
      const response = await api.get('/payments/toss/fail', {
        params: {
          code: error.code,
          message: error.message,
          orderId: error.orderId
        }
      });

      console.log('Payment fail response:', response.data);
      alert(`결제 실패: ${error.message}`);
    }
  } catch (error) {

  } finally {
    await refreshReservations();
  }
};

const refreshReservations = async () => {
  reservationPage.value = 1;
  await fetchReservations();
};

const getCouponTypeText = (type) => {
  const types = {
    'LIMIT': '한정 쿠폰',
    'ALL': '무제한 쿠폰'
  };
  return types[type] || type;
};

watch(activeSection, (newSection) => {
  if (newSection === 'user') fetchUserInfo();
  else if (newSection === 'coupons') fetchCoupons();
  else if (newSection === 'reservations') fetchReservations();
  else if (newSection === 'payments') fetchPayments();
});

const getPaymentStatusText = (status) => {
  const statuses = {
    'PAID': '결제 완료',
    'CANCELLED': '결제 취소'
  };
  return statuses[status] || status;
};
const getPaymentStatusClass = (status) => {
  return {
    'status-paid': status === 'PAID',
    'status-cancelled': status === 'CANCELLED'
  };
};

const getReservationStatusText = (status) => {
  const statuses = {
    'PENDING': '결제 대기중',
    'CONFIRMED': '예약 완료',
    'CANCELLED': '예약 취소',
    'PAID': '결제 완료'
  };
  return statuses[status] || status;
};
const filteredReservations = computed(() => {
  if (reservationFilter.value === 'pending') {
    return reservations.value.filter(reservation => reservation.status === 'PENDING');
  }
  return reservations.value;
});

const filteredPayments = computed(() => {
  if (paymentFilter.value === 'all') {
    return payments.value;
  }
  return payments.value.filter(payment => payment.payStatus === paymentFilter.value);
});
onMounted(async () => {
  const token = localStorage.getItem('token');
  if (!token) {
    router.push('/login');
    return;
  }

  await fetchData(async () => {
    await fetchUserInfo();
    await loadTossPaymentsSDK();
    await refreshData();

    const { paymentKey, orderId, amount, code, message } = route.query;

    if (paymentKey && orderId && amount) {
      await handlePaymentSuccess(paymentKey, orderId, amount);
    } else if (code && message && orderId) {
      await handlePaymentFail({ code, message, orderId });
    }
  });
});
const sections = [
  {value: 'user', label: '회원 정보'},
  {value: 'coupons', label: '쿠폰'},
  {value: 'reservations', label: '예매 내역'},
  {value: 'payments', label: '결제 내역'}
];
</script>

<style scoped>
.my-page-container {
  min-height: 100vh;
  background-color: #ffffff;
  padding: 2rem;
}

.my-page {
  max-width: 800px;
  margin: 0 auto;
  background-color: #D9A66C;
  border-radius: 8px;
  padding: 2rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

h1 {
  font-size: 1.8rem;
  color: #333;
  text-align: center;
  margin-bottom: 2rem;
}

h2 {
  font-size: 1.4rem;
  color: #333;
  margin-bottom: 1.5rem;
}

.button-container {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 2rem;
}

.button-container button {
  padding: 0.75rem 1.5rem;
  background-color: transparent;
  border: 1px solid #D9A66C;
  border-radius: 4px;
  color: #333;
  cursor: pointer;
  transition: all 0.3s ease;
}

.button-container button.active {
  background-color: #D9A66C;
  color: white;
}

.button-container button:hover {
  background-color: #D9A66C;
  color: white;
}

.section {
  background-color: white;
  border-radius: 8px;
  padding: 1.5rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  color: #333;
}

input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
}

.submit-button {
  width: 100%;
  padding: 0.75rem;
  background-color: #D9A66C;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.submit-button:hover {
  background-color: #c08b50;
}

.coupon-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.coupon-item {
  border: 1px solid #D9A66C;
  border-radius: 8px;
  padding: 1rem;
  background-color: white;
}

.coupon-main {
  margin-bottom: 1rem;
}

.coupon-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.coupon-name {
  font-size: 1.1rem;
  font-weight: bold;
  color: #333;
}

.coupon-code {
  background-color: #D9A66C;
  color: white;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  font-size: 0.9rem;
}

.coupon-details {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.discount-info {
  font-weight: bold;
  color: #D9A66C;
}

.coupon-type {
  color: #666;
  font-size: 0.9rem;
}

.coupon-status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #28a745;
  font-size: 0.9rem;
}

.coupon-status.inactive {
  color: #dc3545;
}

.list-container {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.list-item {
  border: 1px solid #eee;
  border-radius: 12px;
  padding: 1.2rem;
  background-color: white;
}

.item-title {
  font-size: 1.1rem;
  font-weight: 600;
  color: #333;
  display: block;
  margin-bottom: 0.25rem;
}

.item-date {
  color: #666;
  font-size: 0.9rem;
  display: block;
  margin-bottom: 0.5rem;
}

.item-details {
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #666;
}

.item-price {
  font-weight: 500;
  color: #D9A66C;
  font-size: 1.1rem;
}

.payment-button {
  margin-top: 0.75rem;
  padding: 0.5rem 1.5rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.3s ease;
  font-size: 0.9rem;
}

.payment-status {
  font-size: 0.9rem;
  padding: 0.4rem 0.8rem;
  border-radius: 15px;
  margin-left: 1rem;
}

.status-paid {
  background-color: #e8f5e9;
  color: #2e7d32;
}

.status-cancelled {
  background-color: #ffebee;
  color: #c62828;
}

.reservation-status {
  margin-top: 0.5rem;
  color: #666;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  margin-top: 1.5rem;
}

.pagination button {
  padding: 0.5rem 1rem;
  background-color: #D9A66C;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.pagination button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.empty-message {
  text-align: center;
  color: #666;
  padding: 2rem 0;
}

.filter-buttons {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.filter-buttons button {
  padding: 0.5rem 1rem;
  background-color: transparent;
  border: 1px solid #D9A66C;
  border-radius: 4px;
  color: #333;
  cursor: pointer;
  transition: all 0.3s ease;
}

.filter-buttons button.active {
  background-color: #D9A66C;
  color: white;
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(255, 255, 255, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.loading-spinner {
  border: 4px solid rgba(0, 0, 0, 0.1);
  border-left-color: #D9A66C;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
