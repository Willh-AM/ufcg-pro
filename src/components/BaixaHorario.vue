<template>
  <button class="btn btn-primary" v-on:click="exportarHorario()" v-bind:disabled="!this.tableTurmas">
    Export
    <span class="glyphicon glyphicon-circle-arrow-down"></span>
  </button>
</template>

<script>
const calendar = require('ics')
// Extraído de https://stackoverflow.com/a/54437356/6732300
const getProximoDiaDaSemana = (dayNumber, excludeToday = true, refDate = new Date()) => {
  const dayMap = {
    2: 'Mon',
    3: 'Tue',
    4: 'Wed',
    5: 'Thu',
    6: 'Fri'
  }
  const dayName = dayMap[dayNumber]
  if (!dayName) return
  const dayOfWeek = ['sun', 'mon', 'tue', 'wed', 'thu', 'fri', 'sat'].indexOf(dayName.slice(0, 3).toLowerCase())
  if (dayOfWeek < 0) return
  refDate.setHours(0, 0, 0, 0)
  refDate.setDate(refDate.getDate() + !!excludeToday + ((dayOfWeek + 7 - refDate.getDay() - !!excludeToday) % 7))
  return refDate
}

const getFimPeriodo = () => {
  // TODO: Receber isso do componente ou ver uma maneira mais inteligente de fazer isso.
  const { periodo, ano } = ufcg.hoje
  const date = new Date()
  date.setFullYear(ano)
  date.setDate('15') // just because
  date.setMonth(periodo == '1' ? 5 : 11)
  return date
}

const getTableTurmas = async () => {
  const doc = await ufcg.getPaginaHorario()
  return doc && doc.querySelectorAll('table')[0]
}

const getNomeDisciplina = tr => {
  const text = tr.children[1].innerText
  if (ufcg.professor) return text
  return text.split(' - ')[1]
}
// Lembrar que essa não é a tabela Turmas em Curso, mas sim a tabela de horário
const getTurmas = ref => {
  return [...ref.querySelectorAll('tbody tr')].map(tr => {
    return {
      cadeira: getNomeDisciplina(tr),
      horario: tr.children[ufcg.professor ? 3 : 4].innerText
        .split('\n')
        .filter(Boolean)
        .map(entry => {
          const [dia, slot, sala] = entry.split(' ')
          return {
            dia: dia.trim(),
            slot: slot.trim(),
            sala: sala
              .trim()
              .replace(')', '')
              .replace('(', '')
          }
        })
        .reduce((acc, { dia, ...rest }) => {
          if (!acc[dia]) acc[dia] = []
          acc[dia].push(rest)
          return acc
        }, {})
    }
  })
}
// For Google Calendar API
const signIn = auth => {
  console.log(auth)

  // const API_ID =
  // const API_KEY =
}

const montaICS = (table, icalendar) => {
  const cadeiras = getTurmas(table)

  const calendar_events = []
  const fimPeriodo = getFimPeriodo()
  cadeiras.forEach(({ horario, cadeira }) => {
    Object.keys(horario).forEach(dia => {
      const slots = horario[dia]
      slots.forEach(({ slot, sala }) => {
        const [inicio, fim] = slot.split('-')
        const proximaData = getProximoDiaDaSemana(dia).toLocaleDateString('en-US')

        const begin = `${proximaData} ${inicio}`
        const end = `${proximaData} ${fim}`

        const [mm, dd, aa] = proximaData.split('/')
        const [start_h, start_m] = inicio.split(':')
        const [end_h, end_m] = fim.split(':')

        const calendar_event = {
          title: cadeira,
          location: sala,
          // ano, mês, dia, hora, minuto
          start: [aa, mm, dd, start_h, start_m],
          end: [aa, mm, dd, end_h, end_m],
          recurrenceRule: 'FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR;INTERVAL=1;UNTIL=' + aa + mm + dd + 'T030000Z'
        }
        calendar_events.push(calendar_event)
      })
    })
  })

  const { error, value } = calendar.createEvents(calendar_events)

  if (error) {
    console.log(error)
    return
  }
  // Importante, não formatar em utf-8
  // download(value, 'horario.ics', 'text/calendar;')
}

const irParaAjuda = () => {
  const win = window.open('https://gist.github.com/lucis/1243ac0bd195647141c7d7f5d3f2691a', '_blank')
  win.focus()
}

function download(data, filename, type) {
  var file = new Blob([data], { type: type })
  if (window.navigator.msSaveOrOpenBlob) window.navigator.msSaveOrOpenBlob(file, filename)
  else {
    var a = document.createElement('a'),
      url = URL.createObjectURL(file)
    a.href = url
    a.download = filename
    document.body.appendChild(a)
    a.click()
    setTimeout(function() {
      document.body.removeChild(a)
      window.URL.revokeObjectURL(url)
    }, 0)
  }
}

export default {
  name: 'BaixaHorario',
  props: ['table'],
  data() {
    return {
      turmas: [],
      tableTurmas: null,
      calendar: null
    }
  },
  methods: {
    exportarHorario() {
      montaICS(this.tableTurmas, this.calendar)
      signIn(this.$root.credentials)

      irParaAjuda()
    }
  },
  mounted() {
    if (this.table) {
      this.tableTurmas = this.table
    } else {
      const table = document.querySelector('table')
      const ths = table && table.querySelectorAll('th')
      const temTabela = ths && [...ths].some(th => th.innerText.includes('Disciplina'))
      if (temTabela) {
        this.tableTurmas = table
      } else {
        // TODO: Não funciona pra professores (não sei se precisa)
        getTableTurmas().then(table => {
          this.tableTurmas = table
        })
      }
    }
  }
}
</script>

<style></style>
